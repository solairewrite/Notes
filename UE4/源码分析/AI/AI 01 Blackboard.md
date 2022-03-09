# AI 01 Blackboard
## 使用黑板
> 1.使用黑板资产并创建黑板组件(if null) AAIController::UseBlackboard

```
bool AAIController::UseBlackboard(UBlackboardData* BlackboardAsset, UBlackboardComponent*& BlackboardComponent)

1.创建黑板组件(if null) NewObject<UBlackboardComponent>

2.调用AAIController::InitializeBlackboard初始化黑板组件,使用传入的BlackboardAsset
```

> 2.初始化黑板 AAIController::InitializeBlackboard

```
bool AAIController::InitializeBlackboard(UBlackboardComponent& BlackboardComp, UBlackboardData& BlackboardAsset)

1.调用UBlackboardComponent::InitializeBlackboard,来使用传入的BlackboardAsset

2.将SelfActor设置为GetPawn()
FName FBlackboard::KeySelf = TEXT("SelfActor");

3.调用蓝图函数OnUsingBlackBoard,自定义初始化黑板数据的逻辑可以写在这里
```

> 3.使用黑板资产,初始化所有key的值 UBlackboardComponent::InitializeBlackboard

```
bool UBlackboardComponent::InitializeBlackboard(UBlackboardData& NewAsset)

1.BlackboardAsset = &NewAsset

2.调用UBlackboardComponent::InitializeParentChain,递归设置每个基类的FirstKeyID,即该基类的所有基类的key数量

3.遍历所有的key,更新每个key的偏移ValueOffsets和所有key的内存块大小ValueMemory(以0填充)

4.为每个key创建实例,存入TArray<UBlackboardKeyType*> KeyInstances

5.遍历每个key,调用UBlackboardKeyType::InitializeKey初始化
5.1调用GetKeyRawData,根据KeyID来获取内存位置
5.2如果创建实例,NewObject<UBlackboardKeyType>,存入KeyInstances中
```

## 其他
> 黑板组件是否兼容黑板资产

```
bool UBlackboardComponent::IsCompatibleWith(UBlackboardData* TestAsset)

如果黑板组件自身的黑板资产,继承自传入的TestAsset,或某个层级的父类所有keys相同,那么就兼容
```