# GameFeatures_04_AddComponent
## 目录
- [GameFeatures_04_AddComponent](#gamefeatures_04_addcomponent)
    - [目录](#目录)
    - [参考](#参考)
    - [概述](#概述)
    - [AddReceiver 将一个Actor作为组件的接受者](#addreceiver-将一个actor作为组件的接受者)
    - [CreateComponentOnInstance 为一个Actor实例创建组件](#createcomponentoninstance-为一个actor实例创建组件)
    - [OnGameFeatureActivating 按照配置为Actor添加组件](#ongamefeatureactivating-按照配置为actor添加组件)
    - [AddComponentRequest 缓存ActorClass,并为Actor实例创建组件](#addcomponentrequest-缓存actorclass并为actor实例创建组件)
    - [内置的组件和Actor](#内置的组件和actor)

## 参考
知乎@大钊 [《InsideUE5》GameFeatures架构（五）AddComponents](https://zhuanlan.zhihu.com/p/492893002)  

## 概述
开发者配置`TArray<FGameFeatureComponentEntry> ComponentList`来定义为哪些Actor添加哪些组件  

GF激活时会调用OnGameFeatureActivating(),遍历ComponentList,将ActorClass和要添加的组件加入ReceiverClassToComponentClassMap  
通过迭代器,根据ActorClass获取所有的Actor实例,对于已经Spawn的Actor可以直接添加组件  

Actor::BeginPlay()中手动调用AddReceiver,会检查ReceiverClassToComponentClassMap是否含有ActorClass  
如果有,说明GF已激活(BeginPlay表示Actor已经是实例了),可以直接为Actor实例添加组件  

## AddReceiver 将一个Actor作为组件的接受者
```
void UGameFrameworkComponentManager::AddReceiverInternal(AActor* Receiver)
{
    for (UClass* Class = Receiver->GetClass(); Class && Class != AActor::StaticClass(); Class = Class->GetSuperClass())
	{
		FComponentRequestReceiverClassPath ReceiverClassPath(Class);
		if (TSet<UClass*>* ComponentClasses = ReceiverClassToComponentClassMap.Find(ReceiverClassPath))
		{
            // Map中存在,说明Action的激活发生在SpawnActor之前,此时新生成的Actor可以直接添加组件
			for (UClass* ComponentClass : *ComponentClasses)
			{
				CreateComponentOnInstance(Receiver, ComponentClass);
			}
		}
    }
}
```

## CreateComponentOnInstance 为一个Actor实例创建组件
```
void UGameFrameworkComponentManager::CreateComponentOnInstance(AActor* ActorInstance, TSubclassOf<UActorComponent> ComponentClass)
{
    UActorComponent* NewComp = NewObject<UActorComponent>(ActorInstance, ComponentClass, ComponentClass->GetFName());

    NewComp->RegisterComponent();
}
```

## OnGameFeatureActivating 按照配置为Actor添加组件
在加载GF时,状态机处于Activating状态会调用此函数  

```
UGameFeatureAction_AddComponents::OnGameFeatureActivating()
{
    AddToWorld(WorldContext, Handles);
}

void UGameFeatureAction_AddComponents::AddToWorld(const FWorldContext& WorldContext, FContextHandles& Handles)
{
    if (UGameFrameworkComponentManager* GFCM = UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance))
    {
        for (const FGameFeatureComponentEntry& Entry : ComponentList)
        {
            TSubclassOf<UActorComponent> ComponentClass = Entry.ComponentClass.LoadSynchronous();

            // 句柄的析构函数中会调用RemoveComponentRequest
            Handles.ComponentRequestHandles.Add(GFCM->AddComponentRequest(Entry.ActorClass, ComponentClass));
        }
    }
}
```

## AddComponentRequest 缓存ActorClass,并为Actor实例创建组件
```
TSharedPtr<FComponentRequestHandle> UGameFrameworkComponentManager::AddComponentRequest(const TSoftClassPtr<AActor>& ReceiverClass, TSubclassOf<UActorComponent> ComponentClass)
{
    FComponentRequestReceiverClassPath ReceiverClassPath(ReceiverClass);
    UClass* ComponentClassPtr = ComponentClass.Get();

    // 判断组件只会加一次的逻辑,这里省略

    TSet<UClass*>& ComponentClasses = ReceiverClassToComponentClassMap.FindOrAdd(ReceiverClassPath);
    ComponentClasses.Add(ComponentClassPtr);

    if (UClass* ReceiverClassPtr = ReceiverClass.Get())
    {
        for (TActorIterator<AActor> ActorIt(LocalWorld, ReceiverClassPtr); ActorIt; ++ActorIt)
        {
            // 如果Actor已经Spawn,直接添加组件
            if (ActorIt->IsActorInitialized())
            {
                CreateComponentOnInstance(*ActorIt, ComponentClass);
            }
        }
    }
}
```

## 内置的组件和Actor
Engine/Plugins/ModularGameplay模块提供了组件,如UPawnComponent  
Lyra/Plugins/ModularGameplayActors模块提供了Actor,如AModularCharacter  
