# GAS 03 GA 02
## K2_CommitAbility
检测费用,CD,如果满足,就消耗费用  

+ 检测是否当前有CooldownGameplayEffectClass
里面的GrantedTags,如果有,说明还在CD  

+ 检测GE属性值能否修改,CostGameplayEffectClass
遍历Modifiers,如果修改后的值<0,说明费用不够  

![](../../../图片/UE4/GAS/技能费用和CD.png)
