# GAS_05_Target
## Character初始化时
1. 绑定ASC的输入确认事件到玩家输入
```
void UAbilitySystemComponent::BindAbilityActivationToInputComponent(UInputComponent* InputComponent, FGameplayAbilityInputBinds BindInfo)
{
    AB.ActionDelegate.GetDelegateForManualSet().BindUObject(this, &UAbilitySystemComponent::LocalInputConfirm);
}
```

## 按下R键开始索敌
1. `UAbilityTask_WaitTargetData::BeginSpawningActor`, `UAbilityTask_WaitTargetData::FinishSpawningActor`这应该是蓝图自动创建的函数

2. `BeginSpawningActor`时  

(1) Task节点绑定AbilityTargetActor的TargetDataReady代理
```
void UAbilityTask_WaitTargetData::InitializeTargetActor(AGameplayAbilityTargetActor* SpawnedActor) const
{
    SpawnedActor->TargetDataReadyDelegate.AddUObject(const_cast<UAbilityTask_WaitTargetData*>(this), &UAbilityTask_WaitTargetData::OnTargetDataReadyCallback);
}
```

(2) Task节点绑定ASC服务器收到客户端RPC过来的TargetData代理
```
void UAbilityTask_WaitTargetData::RegisterTargetDataCallbacks()
{
    AbilitySystemComponent->AbilityTargetDataSetDelegate(SpecHandle, ActivationPredictionKey ).AddUObject(this, &UAbilityTask_WaitTargetData::OnTargetDataReplicatedCallback);
}
```

3. `FinishSpawningActor`时,TargetActor绑定ASC确认事件
```
void AGameplayAbilityTargetActor::BindToConfirmCancelInputs()
{
    ASC->GenericLocalConfirmCallbacks.AddDynamic(this, &AGameplayAbilityTargetActor::ConfirmTargeting);
}
```

## 按下左键释放技能
### 客户端
```
// 每帧调用
void UPlayerInput::ProcessInputStack(const TArray<UInputComponent*>& InputComponentStack, const float DeltaTime, const bool bGamePaused)
{
    static TArray<FDelegateDispatchDetails> NonAxisDelegates;

    for (const FDelegateDispatchDetails& Details : NonAxisDelegates)
    {
        Details.ActionDelegate.Execute(Details.Chord.Key);
    }
}

void UAbilitySystemComponent::LocalInputConfirm()
{
	FAbilityConfirmOrCancel Temp = GenericLocalConfirmCallbacks;
	Temp.Broadcast();
}

void AGameplayAbilityTargetActor::ConfirmTargeting()
{
    ConfirmTargetingAndContinue();
}

void AGameplayAbilityTargetActor_Trace::ConfirmTargetingAndContinue()
{
    TargetDataReadyDelegate.Broadcast(Handle);
}

void UAbilityTask_WaitTargetData::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& Data)
{
    AbilitySystemComponent->CallServerSetReplicatedTargetData(GetAbilitySpecHandle(), GetActivationPredictionKey(), Data, ApplicationTag, AbilitySystemComponent->ScopedPredictionKey);

    // 客户端蓝图收到事件
    ValidData.Broadcast(Data);
}

void UAbilitySystemComponent::CallServerSetReplicatedTargetData(FGameplayAbilitySpecHandle AbilityHandle, FPredictionKey AbilityOriginalPredictionKey, const FGameplayAbilityTargetDataHandle& ReplicatedTargetDataHandle, FGameplayTag ApplicationTag, FPredictionKey CurrentPredictionKey)
{
    ServerSetReplicatedTargetData(AbilityHandle, AbilityOriginalPredictionKey, ReplicatedTargetDataHandle, ApplicationTag, CurrentPredictionKey);
}
```

### 服务器
```
void UAbilitySystemComponent::ServerSetReplicatedTargetData_Implementation(FGameplayAbilitySpecHandle AbilityHandle, FPredictionKey AbilityOriginalPredictionKey, const FGameplayAbilityTargetDataHandle& ReplicatedTargetDataHandle, FGameplayTag ApplicationTag, FPredictionKey CurrentPredictionKey)
{
    ReplicatedData->TargetSetDelegate.Broadcast(ReplicatedTargetDataHandle, ReplicatedData->ApplicationTag);
}

void UAbilityTask_WaitTargetData::OnTargetDataReplicatedCallback(const FGameplayAbilityTargetDataHandle& Data, FGameplayTag ActivationTag)
{
    // 服务器蓝图收到事件
    ValidData.Broadcast(MutableData);
}
```