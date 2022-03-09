# UE4反射代码 09 加载CoreUObject
在`EngineTick`之前会先调用`EnginePreInit`,加载CoreUObject模块  

+ LaunchEngineLoop.cpp
```
bool FEngineLoop::LoadCoreModules() // FEngineLoop::PreInitPreStartupScreen 2035
{
    return FModuleManager::Get().LoadModule(TEXT("CoreUObject")) != nullptr;
}
```

+ CoreNative.cpp
```
class FCoreUObjectModule : public FDefaultModuleImpl
{
    virtual void StartupModule() override
    {
        // 遍历static收集的类,2497个,调用StaticClass()
        UClassRegisterAllCompiledInClasses();

        // 绑定初始化逻辑的代理,下面很快用到
    	void InitUObject();
		FCoreDelegates::OnInit.AddStatic(InitUObject);
    }
}
```

+ ObjectBase.cpp
```
void UClassRegisterAllCompiledInClasses()
{
    // 在.gen.cpp中的IMPLEMENT_CLASS里面,声明static TClassCompiledInDefer
    // 将这个类存入 GetDeferredClassRegistration()
   	TArray<FFieldCompiledInInfo*>& DeferredClassRegistration = GetDeferredClassRegistration();
	for (const FFieldCompiledInInfo* Class : DeferredClassRegistration)
	{
        // 调用TClass::StaticClass();
		UClass* RegisteredClass = Class->Register();
	} 
}
```

+ LaunchEngineLoop.cpp
```
bool FEngineLoop::AppInit( ) // FEngineLoop::PreInitPreStartupScreen 2180
{
    // 调用InitUObject()
    FCoreDelegates::OnInit.Broadcast();
}
```

+ Obj.cpp
```
void InitUObject()
{
    // ProcessNewlyLoadedUObjects会经常调用
    // ProcessLoadedObjectsEvent ProcessLoadedObjectsCallback;
    FModuleManager::Get().OnProcessLoadedObjectsCallback().AddStatic(ProcessNewlyLoadedUObjects);

    // Object初始化
    StaticUObjectInit();
}

void StaticUObjectInit()
{
    UObjectBaseInit(); // 继续转发

    // 创建全局临时包
    // static UPackage* GObjTransientPkg = NULL;
    // NewObject表示此时UObject系统已经创建成功
    GObjTransientPkg = NewObject<UPackage>(nullptr, TEXT("/Engine/Transient"), RF_Transient);
    // 临时包不会被释放
	GObjTransientPkg->AddToRoot();
}

// UObjectGlobals.h
// 这里理解为,默认创建的对象,都放在临时包里面
template< class T >
T* NewObject(UObject* Outer = (UObject*)GetTransientPackage())


// UObject初始化的最终阶段,所有自动注册的object都加入到了主要的数据结构中
void UObjectBaseInit()
{
    // 初始化对象分配器
    // FUObjectAllocator GUObjectAllocator;
	GUObjectAllocator.AllocatePermanentObjectPool(SizeOfPermanentObjectPool);

    //初始化对象管理数组
    // FUObjectArray GUObjectArray;
	GUObjectArray.AllocateObjectPool(MaxUObjects, MaxObjectsNotConsideredByGC, bPreAllocateUObjectArray);

    // 初始化Package(uasset)的异步加载线程
	void InitAsyncThread();
	InitAsyncThread();

	// 单例bool,表示UObject系统初始化完毕,用来判断对象系统是否可用
	Internal::GetUObjectSubsystemInitialised() = true;

    // 处理注册项
	UObjectProcessRegistrants();
}
```

+ UObjectBase.cpp
```
static void UObjectProcessRegistrants()
{
    // 在StaticClass()中,会将所有类加入全局的链
    // static FPendingRegistrant* GFirstPendingRegistrant = NULL; // 全局链表头
    // 这里将链表中的数据存入数组中
    TArray<FPendingRegistrant> PendingRegistrants;
	DequeuePendingAutoRegistrants(PendingRegistrants);

    for(int32 RegistrantIndex = 0;RegistrantIndex < PendingRegistrants.Num();++RegistrantIndex)
	{
		const FPendingRegistrant& PendingRegistrant = PendingRegistrants[RegistrantIndex];

		UObjectForceRegistration(PendingRegistrant.Object, false); // 真正的注册

		// 注册可能导致新的注册项产生(加载其他模块),也加入数组中
		DequeuePendingAutoRegistrants(PendingRegistrants);
	}
}


void UObjectForceRegistration(UObjectBase* Object, bool bCheckForModuleRelease)
{
    // 获取先前在在StaticClass()中,加入的注册信息
	TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();

	FPendingRegistrantInfo* Info = PendingRegistrants.Find(Object);
	if (Info)
	{
		const TCHAR* PackageName = Info->PackageName;
		const TCHAR* Name = Info->Name;
		PendingRegistrants.Remove(Object);
        // 延迟注册
		Object->DeferredRegister(UClass::StaticClass(),PackageName,Name);
	}
}

// 将一个预注册的类转为实际注册的类,并加入全局数组中
void UObjectBase::DeferredRegister(UClass *UClassStaticClass,const TCHAR* PackageName,const TCHAR* InName)
{
	// 创建Package
	UPackage* Package = CreatePackage(PackageName);
    
    // 设置Outer到Package
	OuterPrivate = Package;

	// 设置属于的UClass*类型
	ClassPrivate = UClassStaticClass;

	// 设置对象的名字NamePrivate,加入全局数组中 FUObjectArray GUObjectArray;
	AddObject(FName(InName), EInternalObjectFlags::None);
}
```

+ UObject.h
```
class COREUOBJECT_API UObjectBase
{
    // 对象的类型
	UClass* ClassPrivate;

    // 对象的从属关系,在哪个Package
	UObject* OuterPrivate;
}
```