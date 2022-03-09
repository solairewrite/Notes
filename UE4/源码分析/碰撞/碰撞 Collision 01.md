# 碰撞 Collision 01
## 自定义碰撞通道
Project Settings -> Object Channels,中可自定义碰撞通道  
Project Settings -> Trace Channels,中可自定义射线检测通道  

两者都对应 ECollisionChannel 枚举,从 ECC_GameTraceChannel1 开始  
在ini中用`bTraceType=False`来区分是碰撞还是射线检测  

+ 两者(在一个碰撞组件中)分别对应的字段  
```
// Object Channels
Collision Presets -> Object Type: 自身碰撞通道
Collision Presets -> Collision Responses -> Object Responses: 对其它碰撞通道的响应

// Trace Channels
Collision Presets -> Collision Responses -> Trace Responses: 对其它射线检测的响应
```

比如自己加了一条"我的碰撞通道",配置文件就会自动修改  
```
// DefaultEngine.ini
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,DefaultResponse=ECR_Block,bTraceType=False,bStaticObject=False,Name="我的碰撞通道")
```

```
// EngineTypes.h
enum ECollisionChannel
{
    // 引擎内置通道
    ECC_WorldStatic UMETA(DisplayName="WorldStatic"),
	ECC_WorldDynamic UMETA(DisplayName="WorldDynamic"),
	ECC_Pawn UMETA(DisplayName="Pawn"),
	// ...
	ECC_Destructible UMETA(DisplayName="Destructible"),

    // 引擎预留
    ECC_EngineTraceChannel1 UMETA(Hidden),
    // ...
	ECC_EngineTraceChannel6 UMETA(Hidden),

    // 开发者自定义通道
    ECC_GameTraceChannel1 UMETA(Hidden),
	ECC_GameTraceChannel2 UMETA(Hidden),
    // ...
    ECC_GameTraceChannel18 UMETA(Hidden),
}
```

## 碰撞通道的响应
```
// EngineTypes.h
struct FCollisionResponseContainer
{
    // 记录了(一个ECollisionChannel)对ECollisionChannel中定义的每个通道的响应
    uint8 EnumArray[32];

    ECollisionResponse GetResponse(ECollisionChannel Channel) const 
    {
        return (ECollisionResponse)EnumArray[Channel];
    }
}

enum ECollisionResponse
{
	ECR_Ignore UMETA(DisplayName="Ignore"),
	ECR_Overlap UMETA(DisplayName="Overlap"),
	ECR_Block UMETA(DisplayName="Block"),
	ECR_MAX,
};
```

## 预设碰撞
蓝图中组件可设置的属性  
猜测是ShowOnlyInnerProperties元属性导致内部的属性可以直接暴露在蓝图中  
```
// PrimitiveComponent.h
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Collision, meta=(ShowOnlyInnerProperties, SkipUCSModifiedProperties))
	FBodyInstance BodyInstance;

struct FBodyInstance : public FBodyInstanceCore
{
    UPROPERTY(EditAnywhere, Category=Custom)
	TEnumAsByte<enum ECollisionChannel> ObjectType;

    UPROPERTY(EditAnywhere, Category=Custom)
	TEnumAsByte<ECollisionEnabled::Type> CollisionEnabled;

    UPROPERTY(EditAnywhere, Category = Custom)
	struct FCollisionResponse CollisionResponses;

    UPROPERTY(EditAnywhere, Category=Custom)
	FName CollisionProfileName;
}
```

预设在 Project Settings -> Preset 中进行配置  
对应DefaultEngine.ini中的EditProfiles数组  
```
// CollisionProfile.h
class UCollisionProfile : public UDeveloperSettings
{
    UPROPERTY(globalconfig)
	TArray<FCustomProfile>  EditProfiles;
}

struct FCustomProfile
{	
	FName Name;

	TArray<FResponseChannel>	CustomResponses;
};

// EngineTypes.h
struct FResponseChannel
{
    // 对应ECollisionChannel的展示名
    FName Channel;

    TEnumAsByte<enum ECollisionResponse> Response;
}
```

## 转换函数
这里我理解`ETraceTypeQuery`, `EObjectTypeQuery`其实都是占用的`ECollisionChannel`枚举  
3个枚举的长度全都是32  ,但是前两个枚举进行了重新排序  
实测,`EObjectTypeQuery::ObjectTypeQuery9`是组件 Collision Presets -> Collision Responses -> Object Responses 从上向下数第9个  
在`ECollisionChannel`中对应的是另一个int值  

```
// EngineTypes.h

enum ETraceTypeQuery

enum EObjectTypeQuery

class UEngineTypes : public UObject
{
    static ECollisionChannel ConvertToCollisionChannel(ETraceTypeQuery TraceType);

    static ECollisionChannel ConvertToCollisionChannel(EObjectTypeQuery ObjectType);

    static EObjectTypeQuery ConvertToObjectType(ECollisionChannel CollisionChannel);

    static ETraceTypeQuery ConvertToTraceType(ECollisionChannel CollisionChannel);
}
```