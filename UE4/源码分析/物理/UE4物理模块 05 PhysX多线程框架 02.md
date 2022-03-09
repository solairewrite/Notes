# UE4物理模块 05 PhysX多线程框架 02
### `CpuWorkerThread`
继承于Ps::Thread,PhysX提供的线程运行方案  

CpuWorkerThread是最基础的线程封装,一个CpuWorkerThread代表一个线程  
初始化后,传入CpuDispatche会自动创建一个线程  
tryAcceptJobToLocalQueue用来提交task,放入task队列,execute循环跑,每次检查线程队列里的task运行  

### `Ps::Thread`
`typedef ThreadT<> Thread`  
ThreadT继承于Runnable,Runnable是一个空壳,里面只有execute  
ThreadT的核心成员是`ThreadImpl* mImpl`,CpuDispatcher创建CpuWorkerThread后调用start开启线程  
这时调用`mImpl->start`,最终调用系统api CreateThread,里面会调用execute  

```
// CpuWorkerThread执行task
void Ext::CpuWorkerThread::execute()
{
    mThreadId = getId();

    while (!quitIsSignalled())
    {
        mOwner->resetWakeSignal();

        // 先从本地的线程队列取,
        PxBaseTask* task = TaskQueueHelper::fetchTask(mLocalJobList, mQueueEntryPool);

        // 没有就从全局任务队列拿,或者从其他线程任务队列拿
        if(!task)
            task = mOwner->fetchNextTask();

        if (task)
        {
            mOwner->runTask(*task);
            task->release();
        }
        else
        {
            mOwner->waitForWork();
        }
    }

    quit();
}
```

### `PxTaskManager`
CpuDispatcher用来管理任务,决定哪些任务需要放哪些线程  
TaskManager管理任务依赖,确定Task的运行顺序,决定哪些任务需要提交到CpuDispatcher运行  

```
virtual void setCpuDispatcher(PxCpuDispatcher& ref) = 0;

virtual void resetDependencies() = 0;

virtual void startSimulation() = 0;

virtual void addReference(PxTaskID taskID) = 0;

virtual void decrReference(PxTaskID taskID) = 0;
```

### `PxLightCpuTask`
运行的任务基类,task创建完毕后,无需手动开启,只需要传递taskManager保留引用,建立好依赖关系  
之后所有的运行决策都由TaskManager来做  

主要成员变量:  
+ mRefCount: 引用数  
+ mCont: 自身运行完毕后,要运行的任务  
+ mTm: taskManager的引用  

```
PX_INLINE void setContinuation(PxTaskManager& tm, PxBaseTask* c)
{
    PX_ASSERT( mRefCount == 0 );
    mRefCount = 1;
    mCont = c;
    mTm = &tm;
    if( mCont )
    {
        mCont->addReference();
    }
}
```

task的初始引用为1,当引用为0时才会提交运行,由PxLightCpuTask主动通知TaskManager  

```
// removeReference的时候主动告知taskManager
PX_INLINE void removeReference()
{
    mTm->decrReference(*this);
}

// TaskManager发现引用0，直接提交
void PxTaskMgr::decrReference( PxTaskID taskID )
{
    if( !shdfnd::atomicDecrement( &mTaskTable[ taskID ].mRefCount ) )
    {
        if( dispatchTask( taskID, false ) )
        {
            mGpuDispatcher->finishGroup();
        }
    }
}
```

+ task完成后的通知  

run之后会调用release,就是完成的标志
```
// 主动减去其他的任务依赖,引用为0提交线程运行,如此进行下去
PX_INLINE void release()
{
    if( mCont )
    {
        mCont->removeReference();
    }
}
```

+ 上述多线程raycast小结:  

cpuDispatch开启几个物理线程,task加减引用,taskManager找到引用为0的task提交运行  

## simulate基于CpuDispatcher和TaskManager的多线程运行机制
simulate的统一入口是simulateOrCollide,先做些预备工作,如设定PVD,更新joint,设置buffer标志  
后面开启多线程  

```
// simulateOrCollide
if (simStage == Sc::SimulationStage::eCOLLIDE)
{
    // mSceneCollide -> mCollisionCompletion -> completionTask
    // mSceneCompletion -> null

    mCollisionCompletion.setContinuation(*mTaskManager, completionTask);
    mSceneCollide.setContinuation(&mCollisionCompletion);
    mSceneCompletion.setContinuation(*mTaskManager, NULL);
}
else
{
    // mSceneExecution -> mSceneCompletion -> completionTask

    mSceneCompletion.setContinuation(*mTaskManager, completionTask);
    mSceneExecution.setContinuation(*mTaskManager, &mSceneCompletion);
}

if (simStage == Sc::SimulationStage::eCOLLIDE)
{
    // 运行task
    mCollisionCompletion.removeReference();
    mSceneCollide.removeReference();
}
else
{
    mSceneCompletion.removeReference();
    mSceneExecution.removeReference();
}
```

先初始化mTaskManager,然后setContinuation建立任务依赖关系,最后removeReference把任务运行起来  
completionTask是外部传进来的,可以作为simulate结束的通知  

mSceneExecution里面做的事情  
```
void NpScene::executeScene(PxBaseTask* continuation)
{
    mScene.simulate(mElapsedTime, continuation);
}

void Sc::Scene::simulate(PxReal timeStep, PxBaseTask* continuation)
{
    prepareCollide();
    stepSetupCollide();

    // mCollideStep -> mAdvanceStep -> continuation

    mAdvanceStep.setContinuation(continuation);
    mCollideStep.setContinuation(&mAdvanceStep);

    mAdvanceStep.removeReference();
    mCollideStep.removeReference();
}
```

simalate的运行顺序:  
```
mCollideStep(
    PreRigidBodyNarrowPhase 
    -> RigidBodyNarrowPhase(
        BroadPhase
        -> PostBroadPhase
        -> PostBroadPhase2
        -> PostBroadPhase3
    )
) 
-> mAdvanceStep

碰撞检测(碰撞检测预处理->检测(粗阶段->细化))->碰撞解算
```
