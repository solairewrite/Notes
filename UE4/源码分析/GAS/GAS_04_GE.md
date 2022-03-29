# GAS_04_GE
## 目录
- [GAS_04_GE](#gas_04_ge)
    - [目录](#目录)
    - [简介](#简介)
    - [应用GE](#应用ge)
        - [`MakeEffectContext`](#makeeffectcontext)
        - [`MakeOutgoingSpec`](#makeoutgoingspec)
        - [`ApplyGameplayEffectSpecToSelf`](#applygameplayeffectspectoself)
    - [`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`](#uabilitysystemcomponentapplygameplayeffectspectoself)
        - [免疫](#免疫)
    - [应用非瞬时GE: `FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec`](#应用非瞬时ge-factivegameplayeffectscontainerapplygameplayeffectspec)
        - [GE是如何移除的](#ge是如何移除的)
        - [`FActiveGameplayEffectsContainer::InternalOnActiveGameplayEffectAdded`](#factivegameplayeffectscontainerinternalonactivegameplayeffectadded)
    - [TODO](#todo)

## 简介
`FActiveGameplayEffectsContainer UAbilitySystemComponent::ActiveGameplayEffects`储存所有GE  

各版本应用GE函数的通用入口`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`  

## 应用GE
### `MakeEffectContext`
+ 版本1: `UAbilitySystemComponent::MakeEffectContext()`  

`FGameplayEffectContext::Instigator`设置为OwnerActor  
`FGameplayEffectContext::EffectCauser`设置为AvatarActor  

+ 版本2: `UGameplayAbility::MakeEffectContext()`  

`FGameplayEffectContext::Instigator`设置为OwnerActor  
`FGameplayEffectContext::EffectCauser`设置为AvatarActor  
`FGameplayEffectContext::AbilityCDO`设置为this  
`FGameplayEffectContext::SourceObject`设置为`FGameplayAbilitySpec::SourceObject`  

### `MakeOutgoingSpec`
+ 版本1: `UAbilitySystemComponent::MakeOutgoingSpec()`  

1. 捕捉Attribute,存到`FGameplayEffectSpec::CapturedRelevantAttributes`  

    包括: `UGameplayEffect::Modifiers`属性修改有关的属性(其他略)  

2. 捕捉Tag,存到`FGameplayEffectSpec::CapturedSourceTags`  

    包括: GE自身的Tag,ASC拥有的Tag  

3. 遍历捕捉到的属性,将他们添加到`ASC::ActiveGameplayEffects::AttributeAggregatorMap`
   并复制到`CapturedRelevantAttributes`里面的`AttributeAggregator`  

    捕捉到的属性BaseValue和ASC属性的BaseValue相同  
    `bSnapshot`参数if true,`AttributeAggregator`是根据`AttributeAggregatorMap`复制的  
    else `AttributeAggregator`直接引用`AttributeAggregatorMap`  

+ 版本2: `UGameplayAbility::MakeOutgoingGameplayEffectSpec`  

    和版本1基本相同,`CapturedSourceTags`额外捕捉了GA自身的Tag  

### `ApplyGameplayEffectSpecToSelf`

## `UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`
```
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(const FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
    // 检查免疫,如果免疫,激活失败并广播
    const FActiveGameplayEffect* ImmunityGE=nullptr;
	if (ActiveGameplayEffects.HasApplicationImmunityToSpec(Spec, ImmunityGE))
	{
		OnImmunityBlockGameplayEffectDelegate.Broadcast(Spec, ImmunityGE);
		return FActiveGameplayEffectHandle();
	}

    // 返回的Handle
    FActiveGameplayEffectHandle	MyHandle(INDEX_NONE);

    // 实际应用的GE
    FActiveGameplayEffect* AppliedEffect = nullptr;

    FGameplayEffectSpec* OurCopyOfSpec = nullptr;

    // 对于非瞬时的GE,或客户端预测的瞬时GE
    // bTreatAsInfiniteDuration: 客户端预测的瞬时GE设为true,可能是为了服务器验证失败时回滚
    if (Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant || bTreatAsInfiniteDuration)
    {
        AppliedEffect = ActiveGameplayEffects.ApplyGameplayEffectSpec(Spec, PredictionKey, bFoundExistingStackableGE);
    }
}
```

### 免疫
`UGameplayEffect::GrantedApplicationImmunityTags`里面的RequireTags定义了提供免疫的Tag  
遍历所有激活的GE `ActiveGameplayEffects`,如果新应用的GE的`CapturedSourceTags`包含免疫的Tag,则GE被免疫  
即GE自身的Tag和ASC的Tag,只要包含任意已激活GE的免疫Tag,就会被免疫掉  

被免疫时广播`OnImmunityBlockGameplayEffectDelegate`  

## 应用非瞬时GE: `FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec`
应用一个非瞬时GE,这里不考虑堆栈  

```
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
{
    // 返回的GE
    FActiveGameplayEffect* AppliedActiveGE = nullptr;

    FActiveGameplayEffectHandle NewHandle = FActiveGameplayEffectHandle::GenerateNewHandle(Owner);

    // 这里不确定条件分支流程是否正确
    // 重写了operator new(),在数组的最后一个位置创建新元素,即在所有的激活GE后面添加新GE
    AppliedActiveGE = new(GameplayEffects_Internal) FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);

    FGameplayEffectSpec& AppliedEffectSpec = AppliedActiveGE->Spec;

    // 捕捉Target Tag
    Owner->GetOwnedGameplayTags(AppliedEffectSpec.CapturedTargetTags.GetActorTags());

    // 捕捉Target的ASC的属性值
    // 问题:要捕捉哪些属性是什么时候定义的,可能是和捕捉Modifiers同时发生的,之前跳过了
    AppliedEffectSpec.CaptureAttributeDataFromTarget(Owner);

    // 遍历UGameplayEffect::Modifiers,将修改的值存到FGameplayEffectSpec::Modifiers
    AppliedEffectSpec.CalculateModifierMagnitudes();

    // 定时移除GE
    float DefCalcDuration = 0.f;
	if (AppliedEffectSpec.AttemptCalculateDurationFromDef(DefCalcDuration))
	{
		AppliedEffectSpec.SetDuration(DefCalcDuration, false);
	}
    FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::CheckDurationExpired, AppliedActiveGE->Handle);
	TimerManager.SetTimer(AppliedActiveGE->DurationHandle, Delegate, FinalDuration, false);

    // 设置周期性执行的GE的定时器,这里不考虑

    // 预测客户端绑定代理,在收到服务器拒绝,或服务器认证成功同步GE时,移除客户端预测的GE
    if(预测客户端)
    {
        InPredictionKey.NewRejectOrCaughtUpDelegate(FPredictionKeyEvent::CreateUObject(Owner, &UAbilitySystemComponent::RemoveActiveGameplayEffect_NoReturn, AppliedActiveGE->Handle, -1));
    }

    // TODO
    InternalOnActiveGameplayEffectAdded(*AppliedActiveGE);

    return AppliedActiveGE;
}
```

### GE是如何移除的
1. 创建GE时添加定时器移除GE  
```
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
{
    // 计算GE的持续时间
    // Infinite返回-1,Instant返回0
    float DefCalcDuration = 0.f;
	if (AppliedEffectSpec.AttemptCalculateDurationFromDef(DefCalcDuration))
	{
		AppliedEffectSpec.SetDuration(DefCalcDuration, false);
	}

    // 定时移除GE
    FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::CheckDurationExpired, AppliedActiveGE->Handle);
	TimerManager.SetTimer(AppliedActiveGE->DurationHandle, Delegate, FinalDuration, false);
}
```

### `FActiveGameplayEffectsContainer::InternalOnActiveGameplayEffectAdded`
```
void FActiveGameplayEffectsContainer::InternalOnActiveGameplayEffectAdded(FActiveGameplayEffect& Effect)
{
    const UGameplayEffect* EffectDef = Effect.Spec.Def;

    // 有关GE开关(GE可以被关闭,但它依然处于应用状态)和移除相关的Tag检测,这里不考虑

    // TODO
    // bInvokeGameplayCueEvents = false
    Effect.OwningContainer.AddActiveGameplayEffectGrantedTagsAndModifiers(*this, bInvokeGameplayCueEvents);
}
```

## TODO
`AddActiveGameplayEffectGrantedTagsAndModifiers`  
`Execution`属性捕捉和计算  
瞬时执行的GE  
