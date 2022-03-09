# GAS 04 GE
## C++应用GE
![](../../../图片/UE4/GAS/应用GE.png)

```
FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
EffectContext.AddSourceObject(this);

FGameplayEffectSpecHandle NewHandle = AbilitySystemComponent->MakeOutgoingSpec(DefaultAttributes, GetCharacterLevel(), EffectContext);
if (NewHandle.IsValid())
{
    FActiveGameplayEffectHandle ActiveGEHandle = AbilitySystemComponent->ApplyGameplayEffectSpecToTarget(*NewHandle.Data.Get(), AbilitySystemComponent);
}
```

## 蓝图中应用GE
> `UGameplayAbility::BP_ApplyGameplayEffectToOwner`  

## 蓝图中移除GE
> `void UGameplayAbility::BP_RemoveGameplayEffectFromOwnerWithHandle`  
