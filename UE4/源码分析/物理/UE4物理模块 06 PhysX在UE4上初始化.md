# UE4 物理模块 06 PhysX在UE4上初始化
原文链接: https://zhuanlan.zhihu.com/p/111458039  

## PhysX在UE4的初始化流程
PhysX在UE4的主要功能接口都放在`class FPhysScene_PhysX`  
初始化接口: `FPhysScene_PhysX::InitPhysScene`  
在`UWorld::InitWorld`中调用  
```
void UWorld::InitWorld(const InitializationValues IVS)
{
   	CreatePhysicsScene(WorldSettings);
}

void UWorld::CreatePhysicsScene(const AWorldSettings* Settings)
{
    FPhysScene* NewScene = new FPhysScene(nullptr);
    SetPhysicsScene(NewScene);
}

FPhysScene_PhysX::FPhysScene_PhysX(const AWorldSettings* Settings)
{
    // 创建物理场景
    InitPhysScene(Settings);
}
```

UE4的物理场景设置主要在两个地方:  
1. Edit -> Project Settings,对应`UPhysicsSettings`  
2. Settings -> World Settings,对应`AWorldSettings`  

## `InitPhysScene`
### 设定PhysX的多线程cpu分配器  
```
void FPhysScene_PhysX::InitPhysScene(const AWorldSettings* Settings)
{
	int64 NumPhysxDispatcher = 0;
	FParse::Value(FCommandLine::Get(), TEXT("physxDispatcher="), NumPhysxDispatcher);
	if (NumPhysxDispatcher == 0 && FParse::Param(FCommandLine::Get(), TEXT("physxDispatcher")))
	{
		NumPhysxDispatcher = 4;	// 默认physx 4线程
	}

	// 为任务创建分配器
	if (PhysSingleThreadedMode())
	{
		CPUDispatcher = new FPhysXCPUDispatcherSingleThread();
	}
	else
	{
		if (NumPhysxDispatcher)
		{
			CPUDispatcher = PxDefaultCpuDispatcherCreate(NumPhysxDispatcher);
		}
		else
		{
			CPUDispatcher = new FPhysXCPUDispatcher();
		}

	}
}
```

physx多线程操作以task为基本单位,相互之间通过引用的方式做依赖,然后逐一传给cpuDispatcher进行线程分配  
目前UE4给了3种模式,通过命令行参数设置:  
1. 单线程模式  
2. 使用physx默认的cpuDispatcher,固定4线程  
3. UE4自己实现的cpuDispatcher

### 物理事件通知机制  
```
void FPhysScene_PhysX::InitPhysScene(const AWorldSettings* Settings)
{
    // 创建模拟事件回调
	// physx::PxSimulationEventCallback* FPhysScene_PhysX::SimEventCallback;
	
    // virtual physx::PxSimulationEventCallback* Create(FPhysScene_PhysX* PhysScene) = 0;

	SimEventCallback = SimEventCallbackFactory.IsValid() ? 
        SimEventCallbackFactory->Create(this) : new FPhysXSimEventCallback(this);
}
```

SimEventCallback 包含碰撞,trigger之类的回调  

+ `PxSimulationEventCallback`  
为了接收模拟事件,用户可以实现的接口类  

除了onAdvance(),在调用`PxScene::fetchResults()`或`PxScene::flushSimulation()`  
并且sendPendingReports=true时,事件将被发送  
在运行模拟时(`PxScene::simulate()/::advance()`和`PxScene::fetchResults()`之间)会调用onAdvance()  

不应该在回调事件中修改SDK(软件开发工具包)的状态,尤其是不能创建或摧毁对象  
如果需要更改状态,这些改动应该存入一个缓存中,在simulation阶段之后执行  

查看 `PxScene.setSimulationEventCallback() PxScene.getSimulationEventCallback()`  

+ `PxSimulationEventCallback::onContact()`  
当某些接触事件发生时调用  

这个函数将会为一对actor调用,如果其中一对碰撞形状请求接触通知  
上报事件的请求使用 filter shader/callback 机制  
(查看PxSimulationFilterShader,PxSimulationFilterCallback,PxPairFlag)  

不要维护对被传递对象的引用,因为他们会在这个函数返回时失效  


查看 PxScene.setSimulationEventCallback(), PxSceneDesc.simulationEventCallback  
PxContactPair, PxPairFlag, PxSimulationFilterShader, PxSimulationFilterCallback  

+ `PxSimulationEventCallback::onTrigger()`  
随着当前的触发pair事件调用  

使用`PxShapeFlag::eTRIGGER_SHAPE`标记为触发器的形状会发送事件,根据pair在filter shader中的flag描述  
查看 PxPairFlag, PxSimulationFilterShader  

触发器形状不会为与其他触发器形状的接触事件发送通知,不懂  

查看 PxScene.setSimulationEventCallback(), PxSceneDesc.simulationEventCallback, PxPairFlag  
PxSimulationFilterShader, PxShapeFlag, PxShape.setFlag()  

```
class PxSimulationEventCallback
{
	// 当某些接触事件发生时调用
	// pairHeader: 形状触发接触上报的两个actor的信息
	// pairs: 两个actor的接触对,他们的接触上报被请求
	// nbPairs: 提供的接触对的数量
	// 我的理解: 两个actor接触时,有多个接触对,每个接触对里面又含有多个接触点
	virtual void onContact(
        const PxContactPairHeader& pairHeader, 
        const PxContactPair* pairs, 
		PxU32 nbPairs) = 0;

	// 随着当前的触发对事件调用
	// pairs: 触发对事件
	// count: 触发对事件的数量
    virtual void onTrigger(PxTriggerPair* pairs, PxU32 count) = 0;
}

// 这个类的实例被传给PxSimulationEventCallback.onContact()
struct PxContactPairHeader
{
	// 通知形状对的两个actor,这是physx层面的actor
	// UE4的actor存在 void* PxActor::userData中,
	// class PxRigidActor : public PxActor
	PxRigidActor* actors[2];
}

// 接触上报对的信息
struct PxContactPair
{
	// 组成对的两个形状
	PxShape* shapes[2];
}

struct PxTriggerPair
{
	PxShape* triggerShape; // 被标记为触发器的形状
	PxRigidActor* triggerActor; // 触发器形状附着的actor
	PxShape* otherShape; // 引发触发事件的形状
	PxRigidActor* otherActor; // otherShape附着的actor
}
```
