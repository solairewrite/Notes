# GAS_02_AbilityTask
## 目录
- [GAS_02_AbilityTask](#gas_02_abilitytask)
    - [目录](#目录)
    - [创建AbilityTask的步骤](#创建abilitytask的步骤)
        - [特殊处理SpawnActor类型的AbilityTask](#特殊处理spawnactor类型的abilitytask)
        - [客户端服务器之间发送通用事件](#客户端服务器之间发送通用事件)
            - [GAS内置的通用事件`EAbilityGenericReplicatedEvent`](#gas内置的通用事件eabilitygenericreplicatedevent)
            - [`UAbilitySystemComponent::AbilityTargetDataMap`保存各种通用事件代理](#uabilitysystemcomponentabilitytargetdatamap保存各种通用事件代理)
            - [`UAbilityTask_NetworkSyncPoint`创建服务器和客户端同步点](#uabilitytask_networksyncpoint创建服务器和客户端同步点)
            - [`UAbilityTask_WaitConfirmCancel`监听玩家按下确认键](#uabilitytask_waitconfirmcancel监听玩家按下确认键)

## 创建AbilityTask的步骤
1. 定义代理`DECLARE_DYNAMIC_MULTICAST_DELEGATE`  
将代理标记为`UPROPERTY(BlueprintAssignable)`,作为任务节点的输出针脚  

2. 声明静态工厂函数,用于实例化此任务节点  
调用`NewAbilityTask<>()`来实例化AbilityTask  
此函数的参数为节点的输入针脚,用于设置任务节点的参数  

3. 重写`Activate()`  
此函数自动调用,在这里开始任务逻辑,并绑定代理  

4. 重写`OnDestroy()`  
在这里解绑代理  

### 特殊处理SpawnActor类型的AbilityTask
参考`UAbilityTask_SpawnActor`

可以不重写`Activate()`  
实现`BeginSpawningActor()`和`FinishSpawningActor()`  
这两个函数应该是蓝图自动调用的  

`BeginSpawningActor`应调用`SpawnActorDeferred`来生成Actor,并在这里初始化Actor的变量  
引擎自动生成的字节码会将`expose on spawn`参数暴露给蓝图(源码注释,没亲测)  

`FinishSpawningActor`应调用`AActor::FinishSpawning`,然后调用`EndTask`  

### 客户端服务器之间发送通用事件
#### GAS内置的通用事件`EAbilityGenericReplicatedEvent`
```
namespace EAbilityGenericReplicatedEvent
{
	enum Type
	{	
		GenericConfirm = 0,
		GenericCancel,
		InputPressed,	
		InputReleased,
		GenericSignalFromClient,
		GenericSignalFromServer,
	};
}
```

#### `UAbilitySystemComponent::AbilityTargetDataMap`保存各种通用事件代理
它的键是AbilityHandle和PredictionKey的组合  
它的值是`EAbilityGenericReplicatedEvent`到代理的映射  

1. `CallOrAddReplicatedDelegate`会在`AbilityTargetDataMap`中添加一个事件的代理  
2. `ServerSetReplicatedEvent`会向服务器发送一个通用事件,然后服务器会从`AbilityTargetDataMap`中找到这个事件对应的代理进行广播  
3. `ConsumeGenericReplicatedEvent`会将`AbilityTargetDataMap`中对应事件的代理标记为未触发  

#### `UAbilityTask_NetworkSyncPoint`创建服务器和客户端同步点

通用事件是和AbilityHandle和PredictionKey相关联的,下面参数中的"..."就代指此两者  

1. 向另一端发送信号,并监听来自另一端的信号  
```
void UAbilityTask_NetworkSyncPoint::Activate()
{
    // 如果是客户端
    if (IsPredictingClient())
    {
        // 监听服务器信号
        ReplicatedEventToListenFor = GenericSignalFromServer;

        // 向服务器发送客户端信号
        AbilitySystemComponent->ServerSetReplicatedEvent(GenericSignalFromClient, ...);
    }
    // 如果是服务器
    else if (IsForRemoteClient())
    {
        // 监听客户端信号
        ReplicatedEventToListenFor = GenericSignalFromClient;

        // 向客户端发送服务器信号
        AbilitySystemComponent->ClientSetReplicatedEvent(GenericSignalFromServer, ...);
    }

    // 注册监听信号代理
    AbilitySystemComponent->CallOrAddReplicatedDelegate(
        ReplicatedEventToListenFor, 
        ..., 
        &UAbilityTask_NetworkSyncPoint::OnSignalCallback)
}
```

2. 收到代理后的回调函数  
```
void UAbilityTask_NetworkSyncPoint::OnSignalCallback()
{
    // 消耗信号
    AbilitySystemComponent->ConsumeGenericReplicatedEvent(ReplicatedEventToListenFor, ...);

    // 蓝图执行输出
    OnSync.Broadcast();

    EndTask();
}
```

#### `UAbilityTask_WaitConfirmCancel`监听玩家按下确认键
类似`UAbilityTask_NetworkSyncPoint`,只不过是事件类型变成了`GenericConfirm`  
然后按下确认键只能被客户端检测到,然后RPC通知服务器  
