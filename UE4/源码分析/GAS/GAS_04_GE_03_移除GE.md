# GAS_04_GE_03_移除GE
## 目录
- [GAS_04_GE_03_移除GE](#gas_04_ge_03_移除ge)
    - [目录](#目录)
    - [简介](#简介)
    - [定时移除GE](#定时移除ge)
        - [创建GE时添加定时器移除GE](#创建ge时添加定时器移除ge)
        - [移除GE的定时器`CheckDuration`](#移除ge的定时器checkduration)
        - [服务器移除GE: `InternalRemoveActiveGameplayEffect`](#服务器移除ge-internalremoveactivegameplayeffect)
        - [服务器和客户端移除GE,并清除副作用`RemoveActiveGameplayEffectGrantedTagsAndModifiers`](#服务器和客户端移除ge并清除副作用removeactivegameplayeffectgrantedtagsandmodifiers)
    - [手动移除GE: `BP_RemoveGameplayEffectFromOwnerWithHandle`](#手动移除ge-bp_removegameplayeffectfromownerwithhandle)
    - [客户端,收到服务器同步移除GE](#客户端收到服务器同步移除ge)

## 简介
有持续时间的GE会在创建时设置定时器移除  

## 定时移除GE
### 创建GE时添加定时器移除GE
`UGameplayEffect::DurationMagnitude`定义了GE的持续时间  
在创建GE时会根据它计算出GE的时长FinalDuration,并设置定时器移除GE

```
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
{
    FTimerDelegate Delegate = FTimerDelegate::CreateUObject(..., &UAbilitySystemComponent::CheckDurationExpired);
	
    TimerManager.SetTimer(..., Delegate, FinalDuration, false);
}
```

### 移除GE的定时器`CheckDuration`
`CheckDurationExpired`转发`CheckDuration`  
这里先只考虑移除全部同类型GE的情况  

```
void FActiveGameplayEffectsContainer::CheckDuration(FActiveGameplayEffectHandle Handle)
{
    // 先遍历所有激活的GE,找到要移除的GE
    for (int32 ActiveGEIdx = 0; ActiveGEIdx < GameplayEffects_Internal.Num(); ++ActiveGEIdx)
    {
        FActiveGameplayEffect& Effect = GameplayEffects_Internal[ActiveGEIdx];
        if (Effect.Handle == Handle)
        {
            // 移除周期性函数的定时器
            TimerManager.ClearTimer(Effect.PeriodHandle);

            // StacksToRemove = -1
            InternalRemoveActiveGameplayEffect(ActiveGEIdx, StacksToRemove, false);
        }
    }
}
```

### 服务器移除GE: `InternalRemoveActiveGameplayEffect`
这里不考虑堆栈,不考虑GameplayCue,假定同类型GE全部移除  

```
bool FActiveGameplayEffectsContainer::InternalRemoveActiveGameplayEffect(int32 Idx, int32 StacksToRemove, bool bPrematureRemoval)
{
    FActiveGameplayEffect& Effect = *GetActiveGameplayEffect(Idx);

    InternalOnActiveGameplayEffectRemoved(Effect, ShouldInvokeGameplayCueEvent, GameplayEffectRemovalInfo);

    // 移除定时移除GE的定时器和周期性执行的定时器
    Owner->GetWorld()GetTimerManager().ClearTimer(Effect.DurationHandle);
    Owner->GetWorld()->GetTimerManager().ClearTimer(Effect.PeriodHandle);

    // 从ASC::ActiveGameplayEffects中移除
    GameplayEffects_Internal.RemoveAtSwap(Idx);
}
```

### 服务器和客户端移除GE,并清除副作用`RemoveActiveGameplayEffectGrantedTagsAndModifiers`
`InternalOnActiveGameplayEffectRemoved`标记`FActiveGameplay::IsPendingRemove`为true,然后转发`RemoveActiveGameplayEffectGrantedTagsAndModifiers`  

```
void FActiveGameplayEffectsContainer::RemoveActiveGameplayEffectGrantedTagsAndModifiers(const FActiveGameplayEffect& Effect, bool bInvokeGameplayCueEvents)
{
    // (非周期性GE)恢复属性的CurrentValue
    for(const FGameplayModifierInfo& Mod : Effect.Spec.Def->Modifiers)
    {
        FAggregatorRef* RefPtr = AttributeAggregatorMap.Find(Mod.Attribute);
        RefPtr->Get()->RemoveAggregatorMod(Effect.Handle);
    }

    // 移除GE的GrantedTags和DynamicGrantedTags
    Owner->UpdateTagMap(Effect.Spec.Def->InheritableOwnedTagsContainer.CombinedTags, -1);
	Owner->UpdateTagMap(Effect.Spec.DynamicGrantedTags, -1);

    // 移除免疫,不关注

    // 移除GA,不关注

    // GameplayCue,不关注
}
```

## 手动移除GE: `BP_RemoveGameplayEffectFromOwnerWithHandle`
`UGameplayAbility::BP_RemoveGameplayEffectFromOwnerWithHandle`  
`UAbilitySystemComponent::RemoveActiveGameplayEffect`  
内部转发`InternalRemoveActiveGameplayEffect`,和定时移除GE的流程相同  

## 客户端,收到服务器同步移除GE
`FActiveGameplayEffect::PreReplicatedRemove`  
主要就是调用`RemoveActiveGameplayEffectGrantedTagsAndModifiers`  
