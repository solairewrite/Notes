# GAS 09 AbilityTask
参考 `UAbilityTask_WaitDelay`, `UAbilityTask_WaitInputRelease`  

1. 声明返回自己的静态函数  
```
UFUNCTION(BlueprintCallable, Category="Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
	static UAbilityTask_WaitDelay* WaitDelay(UGameplayAbility* OwningAbility, float Time);
```

在基类`UGameplayTask`的宏`UCLASS(Abstract, BlueprintType, meta = (ExposedAsyncProxy=AsyncTask), config = Game)`  
指明了蓝图结点中指明了对此节点的引用AsyncTask  

2. 声明代理  
```
UPROPERTY(BlueprintAssignable)
	FWaitDelayDelegate	OnFinish;
```

3. 在`Activate()`中绑定代理  

```
DelegateHandle = AbilitySystemComponent->AbilityReplicatedEventDelegate(EAbilityGenericReplicatedEvent::InputReleased, GetAbilitySpecHandle(), GetActivationPredictionKey())
    .AddUObject(this, &UAbilityTask_WaitInputRelease::OnReleaseCallback);
```

4. 广播代理,任务结束  
```
void UAbilityTask_WaitDelay::OnTimeFinish()
{
	if (ShouldBroadcastAbilityTaskDelegates())
	{
		OnFinish.Broadcast();
	}
	EndTask();
}
```