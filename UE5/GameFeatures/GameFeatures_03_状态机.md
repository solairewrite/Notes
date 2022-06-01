# GameFeatures_03_状态机
## 目录
- [GameFeatures_03_状态机](#gamefeatures_03_状态机)
    - [目录](#目录)
    - [参考](#参考)
    - [概述](#概述)
    - [创建状态机](#创建状态机)
        - [UGameFeaturePluginStateMachine::InitStateMachine为所有状态都创建一个实例](#ugamefeaturepluginstatemachineinitstatemachine为所有状态都创建一个实例)
    - [状态机的切换](#状态机的切换)
    - [重要状态](#重要状态)
    - [状态监听](#状态监听)

## 参考
知乎@大钊 [《InsideUE5》GameFeatures架构（四）状态机](https://zhuanlan.zhihu.com/p/484763722)  

## 概述
加载一个GF时,会创建一个状态机管理它的状态,状态机在初始化时,会为每个状态创建一个实例  
将初始状态设置为UnknownStatus,将目标状态设置为Registered(Lyra项目)  
然后状态机就会从当前状态一个一个的切换到下个状态,直到到达目标状态为止  

每个状态中最关键的3个函数是BeginState, UpdateState, EndState  
一般就做一些加载资源和设置下一个状态的工作  

## 创建状态机
UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugin 在加载GF时会为GF创建状态机  
创建完成后会根据.uplugin设置一个目标状态  

```
UGameFeaturePluginStateMachine* UGameFeaturesSubsystem::FindOrCreateGameFeaturePluginStateMachine(const FString& PluginURL)
{
    UGameFeaturePluginStateMachine* NewStateMachine = NewObject<UGameFeaturePluginStateMachine>(this);

    NewStateMachine->InitStateMachine(PluginURL, ...);
}
```

### UGameFeaturePluginStateMachine::InitStateMachine为所有状态都创建一个实例
为所有的状态都创建实例,并将初始状态设置为UnknownStatus,从初始状态开始BeginState()  

```
void UGameFeaturePluginStateMachine::InitStateMachine(const FString& InPluginURL, const FGameFeaturePluginRequestStateMachineDependencies& OnRequestStateMachineDependencies)
{
    CurrentStateInfo.State = EGameFeaturePluginState::UnknownStatus;

    AllStates[(int32)EGameFeaturePluginState::Uninitialized] = MakeUnique<FGameFeaturePluginState_Uninitialized>(StateProperties);
    ...
    AllStates[(int32)EGameFeaturePluginState::Active] = MakeUnique<FGameFeaturePluginState_Active>(StateProperties);

    AllStates[(int32)CurrentStateInfo.State]->BeginState();
}
```

## 状态机的切换
对于Lyra项目,目标状态是Registered  

```
void UGameFeaturePluginStateMachine::SetDestinationState(EGameFeaturePluginState InDestinationState, ...)
{
    // 缓存目标状态
    StateProperties.DestinationState = InDestinationState;

    UpdateStateMachine();
}

void UGameFeaturePluginStateMachine::UpdateStateMachine()
{
    // 不离开do-while循环
    bool bKeepProcessing = false;
    do
    {
        bKeepProcessing = false;

        // 用于储存下一个要切换的状态
        FGameFeaturePluginStateStatus StateStatus;
        // 这里一般会设置好下一个状态
        AllStates[(int32)CurrentState]->UpdateState(StateStatus);

        if (StateStatus.TransitionToState != EGameFeaturePluginState::Uninitialized)
        {
            // 结束当前状态,设置并开始新状态
            AllStates[(int32)CurrentState]->EndState();

            CurrentState = StateStatus.TransitionToState;

            AllStates[(int32)CurrentState]->BeginState();

            bKeepProcessing = true;
        }

    } while (bKeepProcessing)
}
```

## 重要状态
参考UML  

+ Installed: 初始状态,什么都没加载  
+ Registered: 已完成加载dll,依赖的其他插件模块,GF的ini文件,和GameFeatureData  
+ Loaded: 已完成加载GFD中定义的预加载资产  
+ Active: GF已经激活,也已经调用了GFD中所有Action的OnGameFeatureActivating  

## 状态监听
创建IGameFeatureStateChangeObserver,加入UGameFeaturesSubsystem::Observers  
