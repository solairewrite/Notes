# GAS_02_AbilityTask
## 目录
- [GAS_02_AbilityTask](#gas_02_abilitytask)
    - [目录](#目录)
    - [创建AbilityTask的步骤](#创建abilitytask的步骤)
        - [特殊处理SpawnActor类型的AbilityTask](#特殊处理spawnactor类型的abilitytask)
        - [客户端服务器之间发送事件](#客户端服务器之间发送事件)

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
这两个函数应该是蓝图自动创建的  

`BeginSpawningActor`应调用`SpawnActorDeferred`来生成Actor,并在这里初始化Actor的变量  
引擎自动生成的字节码会将`expose on spawn`参数暴露给蓝图(源码注释,没亲测)  

`FinishSpawningActor`应调用`AActor::FinishSpawning`,然后调用`EndTask`  

### 客户端服务器之间发送事件
参考`UAbilityTask_WaitConfirmCancel`,`UAbilityTask_NetworkSyncPoint`  
