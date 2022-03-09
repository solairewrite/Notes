# AI行为树
## 使用行为树
1. NPC_GoblinBP  
`AIControllerClass`: AIC_NPC  
`AutoPossessAI`: Placed in World or Spawned  
2. AIC_NPC  
```
BeginPlay
    -> UseBlackboard // 返回BlcakboardComponent
    -> RunBehaviorTree
```  

## 行为树
### 黑板
`Blackboard`: 为行为树提供变量,在BT的Root节点中设置  
变量类型: `BlackboardKeySelector`  
赋值  
`BTFunctionLibrary::SetBlackboardValueAsObject`  
`BlackboardComponent::SetValueAsBool`  
获取值
`BTFunctionLibrary::GetBlackboardValueAsActor`  

### 行为树自带节点
1. Selector  
从左到右执行子节点,当一个子节点成功时,停止执行,并返回true,如果所有的子节点失败,返回false  
2. Sequence  
从左到右执行,一个失败时,停止执行,一个失败就返回false,所有的都成功返回true  

### 类
创建后即可在BT蓝图节点中右键添加  

---
#### 1. `BTServiceBlueprintBase`
在Selector节点下面添加,  
用于设置黑板值,或设置小怪移动速度等  

重写函数  
+ ReceiveTickAI  
+ ReceiveSearchStartAI  

---
#### 2. `BTDecoratorBlueprintBase`
在Sequence节点上面添加
用于做 `if` 判断  

>内置类  
Blackboard: 判断黑板键值有没有设置,如攻击目标; 或键值大小,如距离  
TimeLimit: 事件限制  

```
Notify Observer: 通知观察者
OnValueChange: 观测的黑板键发生变化时再次评估
OnResultChange: 条件发生变化时再次评估

目前看下来
距离足够近用Value
是否是范围攻击,是否看到玩家用Result

bug: abort-self,notify-value时,如果判断距离大于,就会一卡一卡的
```

```
Observer aborts: 观察中断,不满足条件时怎么办
None: 不终止执行
Self: 结束当前节点和下面的子树
LowerPriority: 结束右面的所有节点
Both: 结束自身和右面
```

重写函数  
+ PerformConditionCheckAI  

---
#### 3. ``
在Sequence节点子树上添加  
执行具体的命令  

>内置类  
RotateToFaceBBEntry: 旋转,面向黑板值中的Actor  
MoveTo: 移动  
Wait: 睡眠  
RunEQSQuery: 查找位置,需要在编辑器设置中勾选AI->EnvironmentQueryingSystem  

重写函数  
+ ReceiveExeceteAI,要调用FinishExecute()  

