# GAS_04_GE的添加和移除

> 当前所有激活的GE  
FActiveGameplayEffectsContainer UAbilitySystemComponent::ActiveGameplayEffects;  

## C++中应用GE
```
// GA中的函数

FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec();

// 动态设置GE的时长
SpecHandle.Data.Get()->SetSetByCallerMagnitude();

ApplyGameplayEffectSpecToOwner();
```

## UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf
```
ActiveGameplayEffects.ApplyGameplayEffectSpec();
```

## FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec
```
// 应用的GE
FActiveGameplayEffect* AppliedActiveGE;
FGameplayEffectSpec& AppliedEffectSpec = AppliedActiveGE->Spec;

// 计算GE时长
float DefCalcDuration = 0.f;
if (AppliedEffectSpec.AttemptCalculateDurationFromDef(DefCalcDuration))
{
    AppliedEffectSpec.SetDuration(DefCalcDuration, false);
}

// 设置定时器,移除GE
FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::CheckDurationExpired, AppliedActiveGE->Handle);

TimerManager.SetTimer(AppliedActiveGE->DurationHandle, Delegate, FinalDuration, false);
```

## FActiveGameplayEffectsContainer::CheckDuration
```
float Duration = Effect.GetDuration();
if (Duration > 0.f && (Effect.StartWorldTime + Duration) < CurrentTime)
{
    InternalRemoveActiveGameplayEffect()
}
```