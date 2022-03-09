# UE4反射代码 08 第一个UClass
## UObject在哪里定义StaticClass
+ NoExportTypes.h
```
// #define CPP 1
// 表示下面的代码不会被编译,但是会被UHT生成反射代码
#if !CPP
...
UCLASS(abstract, noexport)
class UObject
{
	GENERATED_BODY()
...
};
#endif
```

UnrealEngine\Engine\Intermediate\Build\Win64\ShaderCompileWorker\Inc\CoreUObject\NoExportTypes.gen.cpp  
```
IMPLEMENT_CLASS(UObject, 356851929);
```

## StaticClass
+ ObjectMacros.h  
StaticClass()函数实际返回了一个UClass静态指针,单例模式实现  
```
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI  ) \
...
	TRequiredAPI static UClass* GetPrivateStaticClass(); \

	inline static UClass* StaticClass() \
	{ \
		return GetPrivateStaticClass(); \
	} \

#define IMPLEMENT_CLASS(TClass, TClassCrc) \
	UClass* TClass::GetPrivateStaticClass() \
	{ \
		static UClass* PrivateStaticClass = NULL; \
		if (!PrivateStaticClass) \
		{ \
            // Class.cpp中定义的全局函数
			GetPrivateStaticClassBody( \
				PrivateStaticClass, \
...
			); \
		} \
		return PrivateStaticClass; \
	}
```

+ Class.cpp
```
void GetPrivateStaticClassBody(
	...
	UClass*& ReturnClass,
	void(*RegisterNativeFunc)(),
	UClass::ClassConstructorType InClassConstructor,
	UClass::StaticClassFunctionType InSuperClassFn,
	UClass::StaticClassFunctionType InWithinClassFn,
	)
{
...
    // 分配内存
    ReturnClass = (UClass*)GUObjectAllocator.AllocateUObject(sizeof(UClass), alignof(UClass), true);

    // 调用构造函数
    ReturnClass = ::new (ReturnClass) UClass (... InClassConstructor, );

    // 初始化UClass对象
	InitializePrivateStaticClass(
		InSuperClassFn(),
		ReturnClass,
		InWithinClassFn(),
		PackageName,
		Name
		);

	// 注册native函数,即在C++有函数体实现的函数
    // 蓝图中创建的函数和BlueprintImplementableEvent就不是native函数
	RegisterNativeFunc();
}

COREUOBJECT_API void InitializePrivateStaticClass(
	class UClass* TClass_Super_StaticClass,
	class UClass* TClass_PrivateStaticClass,
	class UClass* TClass_WithinClass_StaticClass,
	const TCHAR* PackageName,
	const TCHAR* Name
	)
{
    ...
	// 设置SuperStruct
    TClass_PrivateStaticClass->SetSuperStruct(TClass_Super_StaticClass);
	// 设置outer
	TClass_PrivateStaticClass->ClassWithin = TClass_WithinClass_StaticClass

    // Defer
    TClass_PrivateStaticClass->Register(PackageName, Name);
}
```

+ UObjectBase.cpp  
```
// 注册信息结构
struct FPendingRegistrantInfo
{
	const TCHAR*	Name;
	const TCHAR*	PackageName;
	static TMap<UObjectBase*, FPendingRegistrantInfo>& GetMap()
	{
        // 单例map,使用指针做key,因为对象此时还没有name,需要注册后才有name
		static TMap<UObjectBase*, FPendingRegistrantInfo> PendingRegistrantInfo;
		return PendingRegistrantInfo;
	}
};
// 注册链节点
struct FPendingRegistrant
{
	UObjectBase*	Object;
	FPendingRegistrant*	NextAutoRegister;
};
static FPendingRegistrant* GFirstPendingRegistrant = NULL; // 全局链表头
static FPendingRegistrant* GLastPendingRegistrant = NULL; // 全局链表尾

void UObjectBase::Register(const TCHAR* PackageName,const TCHAR* InName)
{
    // 加入单例map
	TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();

	FPendingRegistrant* PendingRegistration = new FPendingRegistrant(this);
	PendingRegistrants.Add(this, FPendingRegistrantInfo(InName, PackageName));
    
    // 加入全局链表
    if (GLastPendingRegistrant)
    {
        GLastPendingRegistrant->NextAutoRegister = PendingRegistration;
    }
    else
    {
        check(!GFirstPendingRegistrant);
        GFirstPendingRegistrant = PendingRegistration;
    }
    GLastPendingRegistrant = PendingRegistration;
}
```