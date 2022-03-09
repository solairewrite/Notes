# Delay
## 实现原理
向`LatentActionManager::ObjectToActionListMap`中添加这个Actor和延迟函数的映射  
LatentActionManager中每帧减小剩余的时间  
如果小于0,就调用`UObject::ProcessEvent`来执行延迟函数  

+ 向LatentActionManager注册延迟函数  
```
void UKismetSystemLibrary::Delay(UObject* WorldContextObject, float Duration, FLatentActionInfo LatentInfo )
{
    // 向 LatentActionManager.ObjectToActionListMap 中注册这个Actor的延迟函数
    LatentActionManager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, new FDelayAction(Duration, LatentInfo));
}
```

+ Actor每帧检测是否能执行延迟函数
```
void AActor::Tick( float DeltaSeconds )
{
    LatentActionManager.ProcessLatentActions(this, MyWorld->GetDeltaSeconds());
}
```

+ FLatentActionManager每帧检测是否能执行延迟函数
```
// map: [对象 - 函数列表]
TMap<TWeakObjectPtr<UObject>, TSharedPtr<FObjectActions>> ObjectToActionListMap;

void FLatentActionManager::ProcessLatentActions(UObject* InObject, float DeltaTime)
{
    // 从 ObjectToActionListMap 中找Actor的函数列表
    FObjectActions* ObjectActions = GetActionsForObject(InObject)
    TickLatentActionForObject(DeltaTime, ObjectActions->ActionList, InObject);
}

void FLatentActionManager::TickLatentActionForObject(float DeltaTime, FActionList& ObjectActionList, UObject* InObject)
{
    // Action从ObjectActionList中获得
    // 向FLatentResponse::LinksToExecute中添加延迟函数
    Action->UpdateOperation(Response);

    for (FLatentResponse::FExecutionInfo& LinkInfo : Response.LinksToExecute)
    {
        UObject* CallbackTarget = LinkInfo.CallbackTarget.Get();
        CallbackTarget->ProcessEvent(ExecutionFunction, &(LinkInfo.LinkID));
    }
}
```

+ 最终执行函数
```
// ScriptCore.cpp
void UObject::ProcessEvent( UFunction* Function, void* Parms )
```
