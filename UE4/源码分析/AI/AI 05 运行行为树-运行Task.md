# AI 05 运行行为树-运行Task
## 目录
- [AI 05 运行行为树-运行Task](#ai-05-运行行为树-运行task)
    - [目录](#目录)
    - [参考](#参考)
    - [`ProcessPendingExecution`](#processpendingexecution)
        - [1.`ApplySearchData`处理Service和Decorator,如果是Remove类型就调用ReceiveDeactivation.如果是Add类型就调用ReceiveTick](#1applysearchdata处理service和decorator如果是remove类型就调用receivedeactivation如果是add类型就调用receivetick)
            - [`ApplySearchUpdates`对于Remove类型的Service和Decorator,调用蓝图中的反激活事件](#applysearchupdates对于remove类型的service和decorator调用蓝图中的反激活事件)
        - [2.`ExecuteTask`对Task的Service调用ReceiveActivation,对Task调用ReceiveExecute](#2executetask对task的service调用receiveactivation对task调用receiveexecute)

## 参考
@左未 [【图解UE4源码】 其三（三）行为树系统执行任务的流程 执行待执行节点](https://zhuanlan.zhihu.com/p/373310323)

## `ProcessPendingExecution`
1. `ApplySearchData`处理Service和Decorator,如果是Remove类型就调用ReceiveDeactivation.如果是Add类型就调用ReceiveTick
2. `ExecuteTask`对Task的Service调用ReceiveActivation,对Task调用ReceiveExecute

```
void UBehaviorTreeComponent::ProcessPendingExecution()
{
    FBTPendingExecutionInfo SavedInfo = PendingExecution;

    ApplySearchData(SavedInfo.NextTask);

    if (SavedInfo.NextTask)
	{
		ExecuteTask(SavedInfo.NextTask);
	}
}
```

### 1.`ApplySearchData`处理Service和Decorator,如果是Remove类型就调用ReceiveDeactivation.如果是Add类型就调用ReceiveTick
```
// NewActiveNode是PendingExecution.NextTask
void UBehaviorTreeComponent::ApplySearchData(UBTNode* NewActiveNode)
{
    const int32 NewNodeExecutionIndex = NewActiveNode ? NewActiveNode->GetExecutionIndex() : 0;

    ApplySearchUpdates(SearchData.PendingUpdates, NewNodeExecutionIndex);

    for (int32 Idx = 0; Idx < SearchData.PendingUpdates.Num(); Idx++)
    {
        if (UpdateInfo.Mode == EBTNodeUpdateMode::Add)
        {
            const FBehaviorTreeSearchUpdate& UpdateInfo = SearchData.PendingUpdates[Idx];

            // 最终调用UBTAuxiliaryNode::TickNode
            // 对于蓝图Service和Decorator,调用了ReceiveTickAI
            UpdateInfo.AuxNode->WrappedTickNode(...);
        }
    }
}
```

#### `ApplySearchUpdates`对于Remove类型的Service和Decorator,调用蓝图中的反激活事件
```
void UBehaviorTreeComponent::ApplySearchUpdates(const TArray<FBehaviorTreeSearchUpdate>& UpdateList, int32 NewNodeExecutionIndex, bool bPostUpdate)
{
   for (int32 Index = 0; Index < UpdateList.Num(); Index++)
   {
       const FBehaviorTreeSearchUpdate& UpdateInfo = UpdateList[Index];
       FBehaviorTreeInstance& UpdateInstance = InstanceStack[UpdateInfo.InstanceIndex];

        const UBTNode* UpdateNode = UpdateInfo.AuxNode ? (const UBTNode*)UpdateInfo.AuxNode : (const UBTNode*)UpdateInfo.TaskNode;

       if (UpdateInfo.AuxNode)
       {
           uint8* NodeMemory = (uint8*)UpdateNode->GetNodeMemory<uint8>(UpdateInstance);
           if (UpdateInfo.Mode == EBTNodeUpdateMode::Remove)
			{
                // 内部调用UBTAuxiliaryNode::OnCeaseRelevant
                // 对于蓝图Service,调用了ReceiveDeactivationAI
                // 对于蓝图Decorator,调用了ReceiveObserverDeactivatedAI
				UpdateInfo.AuxNode->WrappedOnCeaseRelevant(*this, NodeMemory);
			}
       }
   } 
}
```

### 2.`ExecuteTask`对Task的Service调用ReceiveActivation,对Task调用ReceiveExecute
```
// TaskNode是PendingExecution.NextTask
void UBehaviorTreeComponent::ExecuteTask(UBTTaskNode* TaskNode)
{
    for (int32 ServiceIndex = 0; ServiceIndex < TaskNode->Services.Num(); ServiceIndex++)
    {
        // 对于蓝图Service调用ReceiveActivationAI
        // 对于蓝图Decorator调用ReceiveObserverActivatedAI
        ServiceNode->WrappedOnBecomeRelevant(*this, NodeMemory);
    }

    ActiveInstance.ActiveNode = TaskNode;

    EBTNodeResult::Type TaskResult;
	{
		uint8* NodeMemory = (uint8*)(TaskNode->GetNodeMemory<uint8>(ActiveInstance));

        // 调用UBTTaskNode::ExecuteTask
        // 对于蓝图Task,调用UBTTask_BlueprintBase::ReceiveExecuteAI
		TaskResult = TaskNode->WrappedExecuteTask(*this, NodeMemory);
	}
}
```