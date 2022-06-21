# UPROPERTY
## meta
```
// Categories = "InputTag": GameplayTag只能选InputTag.XXX
UPROPERTY(EditDefaultsOnly, meta = (Categories = "InputTag"))
    FGameplayTag InputTag;

// TitleProperty: 对于结构体数组,在折叠里面的元素时,这个属性会显示为标题
UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities", meta = (TitleProperty = Ability))
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;
```
