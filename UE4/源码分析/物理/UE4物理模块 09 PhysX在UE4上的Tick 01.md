# UE4物理模块 09 PhysX在UE4上的Tick 01
原文链接: https://zhuanlan.zhihu.com/p/111458260  

## PhysX在UE4上的Tick流程
PhysX的tick被包含在 tick group 里(下文简称TG),游戏开始会有初始化流程,中间会做刷新操作  

+ `ETickingGroup` 决定一个tick函数属于哪个TG  
```
enum ETickingGroup
{
    TG_PrePhysics, // 在物理模拟开始前,需要执行的任何对象
    TG_StartPhysics, // 开始物理模拟的特殊TG
    TG_DuringPhysics, // 与物理模拟工作并行的,任何可以执行的对象
    TG_EndPhysics, // 结束物理模拟的特殊TG
    TG_PostPhysics, // 在被执行之前,需要完成刚体和布料模拟的,任何对象
    TG_PostUpdateWork, // 在tick之前,需要update工作做完的任何对象
    TG_LastDemotable, // 任何被降级到最后的容器
    TG_NewlySpawned, // 实际上不是TG的特殊TG,在所有的TG之后,重复运行直到没有新生成的项目
    TG_MAX,
}
```

+ `UWorld::Tick` 中按照顺序调用 `RunTickGroup`  
```
void UWorld::Tick( ELevelTick TickType, float DeltaSeconds )
{
    // 如果物理tick函数没有设置,就将其设置
    SetupPhysicsTickFunctions(DeltaSeconds);

    // 重置到开始的TG
    TickGroup = TG_PrePhysics;

    // 获取单例 FTickTaskManager,调用StartFrame
    // 根据TG,tick指定levels中的动态actors,对于每个TG,这个函数执行一次
    FTickTaskManagerInterface::Get().StartFrame(this, DeltaSeconds, TickType, LevelsToTick);

    {
        // 每一个RunTickGroup的上下文都有复杂的宏
        RunTickGroup(TG_PrePhysics);
    }

    bInTick = false;
    // 确保碰撞检测树已经完全构建,应该在关卡加载完后调用,以确保第一次检测不会特别慢
    // 内部调用 FPhysScene_PhysX::EnsureCollisionTreeIsBuilt
    EnsureCollisionTreeIsBuilt();
    bInTick = true;

    {
        RunTickGroup(TG_StartPhysics);
    }
    {
        // 这里不等待,不过应该一直运行直到idle(英文释义:没有工作的)
        // 在开始运行后期物理事件前,我们不关心是否所有的异步tick都完成
        RunTickGroup(TG_DuringPhysics, false);
    }
    // 在这里设置此项,以便当前的TG在碰撞通知期间是正确的,不过我不确定这是否有必要
    TickGroup = TG_EndPhysics;
    {
        RunTickGroup(TG_EndPhysics);
    }
    {
        RunTickGroup(TG_PostPhysics);
    }
}
```

+ `FStartPhysicsTickFunction` 开始物理tick的tick函数  
```
struct FStartPhysicsTickFunction : public FTickFunction
{
    class UWorld*	Target; // 这个函数属于哪个world

    // 实际执行tick的抽象函数
    virtual void ExecuteTick(float DeltaTime, 
        enum ELevelTick TickType, 
        ENamedThreads::Type CurrentThread, 
        const FGraphEventRef& MyCompletionGraphEvent) override;
}

void FStartPhysicsTickFunction::ExecuteTick()
{
    Target->StartPhysicsSim(); 
}

void UWorld::StartPhysicsSim()
{
    FPhysScene* PhysScene = GetPhysicsScene();

    // 调用 TickPhysScene(PhysicsSubsceneCompletion);
    // 主要计算物理 DeltaTime
    // 与骨骼绑定的Kinematic碰撞盒位置计算
    // 进行 simulate 或 substep simulate
    // 运行 FGraphEventRef PhysicsSubsceneCompletion;
    // 物理场景的完成事件(不是任务),当物理系统完成时触发
    // 里面每个 simulate 都会对应一个 fetchResults,具体看 physx.PxScene.Simulate
    PhysScene->StartFrame();
}
```

### `UWorld::SetupPhysicsTickFunctions` 注册物理接口  
`FStartPhysicsTickFunction UWorld::StartPhysicsTickFunction`  
`FEndPhysicsTickFunction UWorld::EndPhysicsTickFunction`  
```
void UWorld::SetupPhysicsTickFunctions(float DeltaSeconds)
{
    StartPhysicsTickFunction.bCanEverTick = true;
	StartPhysicsTickFunction.Target = this;

    EndPhysicsTickFunction.bCanEverTick = true;
	EndPhysicsTickFunction.Target = this;

    // uint8 UWorld::bShouldSimulatePhysics 如果true,这个world会tick物理模拟
    bool bEnablePhysics = bShouldSimulatePhysics;

    if (bEnablePhysics && !StartPhysicsTickFunction.IsTickFunctionRegistered())
    {
        StartPhysicsTickFunction.TickGroup = TG_StartPhysics;
        StartPhysicsTickFunction.RegisterTickFunction(PersistentLevel);
    }

    if (bEnablePhysics && !EndPhysicsTickFunction.IsTickFunctionRegistered())
    {
        EndPhysicsTickFunction.TickGroup = TG_EndPhysics;
        EndPhysicsTickFunction.RegisterTickFunction(PersistentLevel);
        EndPhysicsTickFunction.AddPrerequisite(this, StartPhysicsTickFunction);
    }
}

// 通过单例 FTickTaskManager,将 FTickFunction 加入 AllEnabledTickFunctions
// FTickTaskLevel* ULevel::TickTaskLevel
// TSet<FTickFunction*> FTickTaskLevel::AllEnabledTickFunctions

void FTickFunction::RegisterTickFunction(ULevel* Level)
{
    FTickTaskManager::Get().AddTickFunction(Level, this);
}
```

### `FTickTaskManager::RunTickGroup`  
```
// 运行一个TG,tick所有的actor和组件
// bBlockTillComplete: 直到所有的tick都完成才return
void UWorld::RunTickGroup(ETickingGroup Group, bool bBlockTillComplete = true)
{
    check(TickGroup == Group); // 应该已经是正确的值了,这里想确保顺序正确
    FTickTaskManagerInterface::Get().RunTickGroup(Group, bBlockTillComplete);
    TickGroup = ETickingGroup(TickGroup + 1); // 新的actors进入下一个TG
}

virtual void FTickTaskManager::RunTickGroup(ETickingGroup Group, bool bBlockTillComplete ) override
{
    // 释放指定TG队列中的ticks,并处理他们
    TickTaskSequencer.ReleaseTickGroup(Group, bBlockTillComplete);

    Context.TickGroup = ETickingGroup(Context.TickGroup + 1); // 新的actors进入下一个TG

    // 我们不处理在异步TG新生成的ticks,他们等待,直到完成异步事情
    if (bBlockTillComplete)
    {
        bool bFinished = false;
        // 通过循环100次,来完成有限递归生成的ticks
        for (int32 Iterations = 0;Iterations < 101; Iterations++)
        {
            int32 Num = 0;
            // 遍历 TArray<FTickTaskLevel*> FTickTaskManager::LevelList;
            // 获取所有新生成的ticks,并处理他们
            for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
            {
                // 将这个level新生成的ticks加入队列,并返回数量
                Num += LevelList[LevelIndex]->QueueNewlySpawned(Context.TickGroup);
            }
            // 如果还存在新生成的ticks就进行处理
            if (Num && Context.TickGroup == TG_NewlySpawned)
            {
                TickTaskSequencer.ReleaseTickGroup(TG_NewlySpawned, true);
            }
            else
            {
                bFinished = true;
                break;
            }
        }
        // 循环了100次之后,还是没完成,剩下的就是难以控制的递归生成
        if (!bFinished)
        {
            for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
            {
                // 如果有无限递归的生成,log并丢弃它们
                LevelList[LevelIndex]->LogAndDiscardRunawayNewlySpawned(Context.TickGroup);
            }
        }
    }
}
```

+ `FTickTaskSequencer`  

是处理实际的tick任务,开始和完成TG的class  

+ `FTickTaskSequencer::ReleaseTickGroup`  

释放指定TG的队列中的ticks,并处理他们  

判断单线程和多线程
单线程调用`FTickTaskSequencer::DispatchTickGroup`  
多线程在其他线程上分配TG,这样的话,游戏线程就可以处理ticks,同时ticks在别得线程排队  

如果阻塞或单线程,直到任务完成前都等待  

对于不阻塞的多线程,因为在另一个任务(物理)占用异步时间,所以应该处理现有的任何东西  

如果单线程就禁止阻塞  



阻塞的实现是通过while里面检测是否完成,如果没完成就重新执行任务  

+ `FTickTaskSequencer::DispatchTickGroup`  

每个TG有高优先度的任务和低优先度的任务,遍历这些任务,调用`TGraphTask::Unlock`  
```
// class FTickTaskSequencer
// 猜测两个索引分别代表开始的TG和结束的TG
TArrayWithThreadsafeAdd<TGraphTask<FTickFunctionTask>*> HiPriTickTasks[TG_MAX][TG_MAX];

TArrayWithThreadsafeAdd<TGraphTask<FTickFunctionTask>*> TickTasks[TG_MAX][TG_MAX];
```

+ `TGraphTask::Unlock`  

一个任务有若干未完成的先决条件,这里-1,如果数量为0,则将任务放入队列中等待执行  

具体通过`FTaskGraphInterface`单例执行`FTaskGraphInterface::QueueTask`  

放入队列中的任务有的可以直接执行,有的要加入队列中,可能是本线程的或其他线程的队列  
