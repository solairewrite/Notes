# CanActivateAbility
尝试激活一个技能时,调用此函数,判断技能能否被激活  
```
// 尝试激活技能
InternalTryActivateAbility()
{
	// 依次检测CD,Cost,Tag,蓝图实现,判断能否激活技能
	// 失败时,引用参数OptionalRelevantTags,会返回UAbilitySystemGlobals中的预定义tag,来标识失败原因
	// 定义在DefaultGame.ini中
	if(!CanActivateAbility(..., OptionalRelevantTags))
	{
		NotifyAbilityFailed(..., OptionalRelevantTags);
		return false;
	}
	
	// 技能激活
	CallActivateAbility();
}
```

## 一,CheckCooldown
+ 获取CD tag  
GetCooldownTags() GA蓝图中设置的CooldownGameplayEffectClass,里面的GrantedTags
+ 判断技能组件是否含有这些tag  

OptionalRelevantTags: Ability.Fail.Cooldown  

## 二,CheckCost
+ 获取GA蓝图中设置的CostGameplayEffectClass  
+ 调用CanApplyAttributeModifiers,遍历GE中的Modifiers修改的FGameplayAttribute  
如果修改类型为Add,并且 CurrentValue + CostValue < 0,返回false

OptionalRelevantTags: Ability.Fail.Cost  

## 三,DoesAbilitySatisfyTagRequirements
在满足下列3点时返回true  
1. GA自身的AbilityTags 没有被 其它的GA的BlockAbilitiesWithTag 所阻挡  
	这些tag存于 FGameplayTagContainer  AbilitySystemComponent::BlockedAbilityTags::ExplicitTags  
2. activating actor/component 没有 ActivationBlockedTags  
3. activating actor/component 有全部的 ActivationRequiredTags  

这里我理解为 activating actor/component 就是玩家自身的 AbilitySystemComponent  

OptionalRelevantTags: Ability.Fail.TagsBlocked (对于上面1和2)  
OptionalRelevantTags: Ability.Fail.TagsMissing (对于上面3)  

## 四,K2_CanActivateAbility
蓝图中判断技能是否可以激活,如果前面的CD,Cost,Tag检测失败,这里不会被执行  
