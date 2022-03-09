# GamePlay架构

##  1. Actor和Component
Actor的坐标内部转发到RootComponent中  
Actor处理Input事件的能力,内部转发到UInputComponent中  
`TSet<UActorComponent*> OwnedComponents;`所有组件  
`USceneComponent* RootComponent;`  
`TArray<UActorComponent*> InstanceComponents;`被实例化的组件  
如果Actor可以放进Level里,就必须实例化RootComponent  

+ USceneComponent  
提供了Transform和SceneComponent的互相嵌套,但是没有渲染和碰撞的能力  

+ Actor之间的父子关系  
通过Component确定父子关系,将一个Actor的RootComponent附着到另一个Actor的RootComponent  
`TArray<AActor*> Children;`  
通过`AActor::AttachToActor()`和`AActor::AttachToComponent()`创建父子关系  

## 2. Level和World
World没有直接引用Level里的Actor,`TActorIterator`(World的Actor迭代器)内部的实现只是遍历Levels来获取所有Actor  
World对Controllers和Pawn保存了引用  
Levels共享World的一个PhysicsScene  

## 3. WorldContext, GameInstance, Engine
WorldContext既负责World之间切换的上下文,也负责Level之间切换的操作信息  
GameInstance里面保存WorldContext和其他整个游戏的信息  
GameInstance不论Level如何切换,还会一直存在,可以存放全局数据  

Object -> Actor + Component -> Level -> World -> WorldContext -> GameInstance -> Engine  

## 4. Pawn
1. 可被Controller控制  
2. PhysicsCollision表示  
3. MovementInout的基本响应接口  

## 5. Controller

## 6. PlayerController和AIController
AIController用`SetFocus()`来控制Pawn视角朝向  

## 7. GameMode和GameState

## 8. Player
`ULocalPlayer* UGameInstance::CreateLocalPlayer()`创建ULocalPlayer  
同时调用`创建ULocalPlayer::SpawnPlayActor`,生成PlayerController  
`AController::InitPlayerState()`创建PlayerState  

## 9. GameInstance
`UGameEngine::Init(IEngineLoop* InEngineLoop)`创建GameInstance  
可以在BaseEngine.ini或DefaultEngine.ini里配置GameInstanceClass  
```
[/Script/EngineSettings.GameMapsSettings]
GameInstanceClass=/Script/Engine.GameInstance
```

+ 玩家存档 USaveGame  
继承自`USaveGame`就可以序列化保存下来  

## 10. 总结
>UObject功能  
+ 反射: 运行时创建Class  
+ 网络同步: RPC调用,属性同步  
+ 序列化: 保存uasset文件,USaveGame存档  
+ 属性编辑器  
+ 垃圾回收  
