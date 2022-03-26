# GAS_04_GE
## 目录
- [GAS_04_GE](#gas_04_ge)
    - [目录](#目录)
    - [简介](#简介)
    - [GE是如何移除的](#ge是如何移除的)

## 简介
`FActiveGameplayEffectsContainer UAbilitySystemComponent::ActiveGameplayEffects`储存所有GE  

各版本应用GE函数的通用入口`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`  

## GE是如何移除的
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