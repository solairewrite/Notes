# AI
## EQS Environment Querying System
向场景中添加 NavMeshBoundsVolume  
在选定位置附近生成一系列测试点,并根据自定义的规则对这些测试点进行评分,返回合适的测试点数组  
### 应用
在指定位置附近生成敌人,距离玩家越远越好  
流程:  
+ 编辑 -> 编辑器偏好设置,勾选 Environment Querying System  
+ 新建: 人工智能 -> 环境查询,命名为 EQS_FindSpawnLoc  
+ 新建: BP -> EnvQueryContext_BlueprintBase,命名为 EQS_BotSpawns  
+ 编辑 EQS_BotSpawns,重写函数 Provide Actors Set -> Get All Actors of Class(Target Point) -> Return Node, 作为生成测试栅格的Actor  
+ 相同的,新建 EQS_AllPlayers, 返回自定义的玩家类  
+ 编辑 EQS_FindSpawnLoc, 从Root拖出 Points: Grid, 设置 Generate Around: EQS_BotSpawns  
+ 在 SimpleGrid 节点,右键添加测试 Distance,设置 Distance to: EQS_AllPlayers  
+ 
+ 在 GameMode 蓝图中,合适的事件后(如BeginPlay),添加节点 Run EQSQuery(EQS_FindSpawnLoc)  
+ -> 绑定事件到 OnQueryFinished,添加符合签名的自定义事件  
+ Switch Query Status,在 Success 的情况下 -> Get Query Results as Locations(Query Instance),查询到的点的数组  

### 测试方式
Distance: 判断到指定类型Actor的距离,Scoring Factor 为负值时,越近越好  
Trace: 射线检测,即判断是否有障碍物  
测试目的可以选择仅用于评分,表示即使没有障碍物,也可以  
Filter and Score && not Hit && Item Height Offset, Context Height Offset 同时设为100  
表示不希望有障碍物,但是它的高度不高时,可通过  

## Behavior Tree 行为树
运行时,单引号( ' )可以显示AI状态  
| 类 | 作用 |
| - | - |
| Behavior Tree | 行为树,实际控制AI的行为,维护一个Blackboard |
| Blackboard | 黑板,用于储存行为树中的变量 |
| EQS | 用于查找位置/物体,找到后可以设为黑板的变量 |
| EnvQueryContext_BlueprintBase | 返回目标物体,用于EQS的测试节点 |
| BTDecorator_BlueprintBase | 返回bool值,位于行为树的Selector节点下方的Sequence节点上方,用于判断能否执行这个Sequence |
| BTTask_BlueprintBase | 自定义行为树事件 |

### Behavior Tree 重要节点
| 节点 | 作用 |
| - | - |
| Root | 根节点,可以指定用于储存变量的Blackboard |
| Selector | 从左到右依次执行节点,如果一个成功了,就返回成功;如果全部失败,则返回失败 |
| Decorator | 位于Selector下方节点的上方,同于判断该节点能否进行 |
| Sequence | 从左到右依次执行节点 |
| Wait | 睡眠 |
| Run EQSQuery | 执行EQS查询,并可以指定Blackboard Key,将查询结果存储于黑板中 |
| Move To | 移动到指定位置,设置Blackboard key来获取位置 |
| Blackboard | 查询Blackboard的属性是否有值,返回bool值,在节点上方判断节点是否运行 |
| Default Focus | 看 |

+ AI 通过蓝图运行行为树  
  self -> Get AIController -> Run Behavior Tree

+ AI BP 设置黑板值
  self -> Get Blackboard -> Set Value as Object  
+ Decorator 重写 Perform Condition Check AI  
+ Decorator中获取黑板中的变量  
  创建节点Get Blackboard Value as Actor,并将Key提升为变量Blackboard Key, 在BT中指定Blackboard Key  
+ EnvQueryContext_BlueprintBase 重写  
  Provide Actors Set  或 Provide Single Actor  
+ Decorator 的观察器终止属性  
  观察器终止: Self, 当Decorator的bool == false时,停止此节点  
  观察器终止: Lower Priority, 当Decorator的bool == true时,开始此节点,停止右面的节点
+ BTTask_BlueprintBase 重写Receive Execute AI
