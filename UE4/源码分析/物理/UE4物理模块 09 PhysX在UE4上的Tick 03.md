# UE4物理模块 09 PhysX在UE4上的Tick 03
+ TG_StartPhysics阶段  

主要进行了simulate,substep,fetchResults  

FStartPhysicsTickFunction::ExecuteTick  
-> UWorld::StartPhysicsSim  
-> FPhysScene_PhysX::StartFrame  

+ TG_EndPhysics阶段  

将一个渲染帧时间里,物理模拟得到的碰撞事件反馈到各个UE的对象上  

FEndPhysicsTickFunction::ExecuteTick  
-> UWorld::FinishPhysicsSim  
-> FPhysScene_PhysX::EndFrame  
-> DispatchPhysNotifications_AssumesLocked  

## FEndPhysicsTickFunction

+ FEndPhysicsTickFunction::ExecuteTick  
调用UWorld::FinishPhysicsSim  
```
void FEndPhysicsTickFunction::ExecuteTick(
    float DeltaTime, 
    enum ELevelTick TickType, 
    ENamedThreads::Type CurrentThread, 
    const FGraphEventRef& MyCompletionGraphEvent)
{
    FPhysScene* PhysScene = Target->GetPhysicsScene();

    FGraphEventArray PhysicsComplete = PhysScene->GetCompletionEvents();
    if (!PhysScene->IsCompletionEventComplete())
    {
        // 不要释放下一个TG,直到物理已经完成并且运行过FinishPhysicsSim

        MyCompletionGraphEvent->DontCompleteUntil(
			FSimpleDelegateGraphTask::CreateAndDispatchWhenReady(
				FSimpleDelegateGraphTask::FDelegate::CreateUObject(Target, &UWorld::FinishPhysicsSim),
				GET_STATID(STAT_FSimpleDelegateGraphTask_FinishPhysicsSim), &PhysicsComplete, ENamedThreads::GameThread
			)
		);
    }
    else
    {
        Target->FinishPhysicsSim();
    }
}
```

+ UWorld::FinishPhysicsSim  
```
void UWorld::FinishPhysicsSim()
{
    PhysScene->EndFrame(LineBatcher);
}
```

+ FPhysScene_PhysX::EndFrame  
```
void FPhysScene_PhysX::EndFrame(ULineBatchComponent* InLineBatcher)
{
    PhysicsSceneCompletion = NULL;

    // 这里物理模拟已经结束

    // 执行碰撞通知事件
    DispatchPhysNotifications_AssumesLocked();

    // debug渲染,batch:批量
    if (InLineBatcher)
	{
		AddDebugLines(InLineBatcher);
	}
}
```

+ SceneCompletionTask  
用来做最后一个fetchResults  

StartFrame里面创建事件SceneCompletionTask,先决条件是PhysicsSubsceneCompletion  
TickPhysScene里面将PhysicsSubsceneCompletion包裹到task中,需要等simulate结束才能回调  

simulate这行代码执行完,很多工作会丢到其他线程  
PhysicsSubsceneCompletion不会立即调用,SceneCompletionTask也不会立即调用  

TG_StartPhysics阶段不会完成,在TG_EndPhysics阶段  
FEndPhysicsTickFunction::ExecuteTick中,执行DontCompleteUntil,等待物理完成一帧模拟才会调用  

SubstepSimulationStart,SubstepSimulationEnd也是在TG_EndPhysics才真正结束  

```
void FPhysScene_PhysX::StartFrame()
{
    TickPhysScene(PhysicsSubsceneCompletion);

    FGraphEventArray MainScenePrerequisites;

    MainScenePrerequisites.Add(PhysicsSubsceneCompletion);

    new (FinishPrerequisites)FGraphEventRef(
        FDelegateGraphTask::CreateAndDispatchWhenReady(
            FDelegateGraphTask::FDelegate::CreateRaw(this, &FPhysScene_PhysX::SceneCompletionTask),
            GET_STATID(STAT_FDelegateGraphTask_ProcessPhysScene_Sync), &MainScenePrerequisites,
            ENamedThreads::GameThread, ENamedThreads::GameThread
        )
    );
}

void FPhysScene_PhysX::TickPhysScene(FGraphEventRef& InOutCompletionEvent)
{
    PhysXCompletionTask* Task = new PhysXCompletionTask(InOutCompletionEvent, ApexScene->getTaskManager());
    ApexScene->simulate(AveragedFrameTime, true, Task, SimScratchBuffer.Buffer, SimScratchBuffer.BufferSize);
}
```
