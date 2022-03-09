# 获取,各种Get
+ 获取组件  
```
MyComponentClass* Comp = Cast<MyComponentClass>
(MyActor->GetComponentByClass(MyComponentClass::StaticClass()));
```
+ 组件获取Actor  
```
GetOwner()
```
+ 获取Character  
```
ACharacter* tPlayer = UGameplayStatics::GetPlayerCharacter(this, 0);
```
+ 获取玩家移动组件  
```
GetMovementComponent()
```
