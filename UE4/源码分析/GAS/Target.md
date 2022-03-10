# Target
## 按下R键开始索敌

1. `UAbilityTask_WaitTargetData::BeginSpawningActor`, `UAbilityTask_WaitTargetData::FinishSpawningActor`这应该是蓝图自动创建的函数

2. Task节点绑定AbilityTargetActor的TargetDataReady代理
```
void UAbilityTask_WaitTargetData::InitializeTargetActor(AGameplayAbilityTargetActor* SpawnedActor) const
{
    SpawnedActor->TargetDataReadyDelegate.AddUObject(const_cast<UAbilityTask_WaitTargetData*>(this), &UAbilityTask_WaitTargetData::OnTargetDataReadyCallback);
}
```

## 按下左键释放技能

TODO: 查看确认按键绑定
```
void UAbilityTask_WaitTargetData::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& Data)
{

}
```