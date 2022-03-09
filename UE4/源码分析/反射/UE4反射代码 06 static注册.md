# UE4反射代码 06 static注册
## 收集 UClass
+ UObjectBase.h  
```
// 用于类延迟注册的基类
struct FFieldCompiledInInfo{}

// 类延迟注册的模板类
template <typename TClass>
struct TClassCompiledInDefer : public FFieldCompiledInInfo
{
	TClassCompiledInDefer(const TCHAR* InName, SIZE_T InClassSize, uint32 InCrc)
	: FFieldCompiledInInfo(InClassSize, InCrc)
	{
        // 向延迟注册队列添加一个类
        // static TMap<FName, FFieldCompiledInInfo*> DeferRegisterClassMap;
		UClassCompiledInDefer(this, InName, InClassSize, InCrc);
	}
	virtual UClass* Register() const override
	{
        // StaticClass()其实是UClass*单例,第一次获取时会赋值
		return TClass::StaticClass();
	}
};
```

+ ObjectMacros.h  
```
// 声明StaticClass()函数
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI  ) \
	TRequiredAPI static UClass* GetPrivateStaticClass(); \
	inline static UClass* StaticClass() \
	{ \
		return GetPrivateStaticClass(); \
	} \

// StaticClass() 实际返回 GetPrivateStaticClass()函数内的静态UClass*单例
#define IMPLEMENT_CLASS(TClass, TClassCrc) \
	static TClassCompiledInDefer<TClass> AutoInitialize##TClass(TEXT(#TClass), sizeof(TClass), TClassCrc); \
	UClass* TClass::GetPrivateStaticClass() \
	{ \
		static UClass* PrivateStaticClass = NULL; \
		if (!PrivateStaticClass) \
		{ \
			GetPrivateStaticClassBody( \
				...
				PrivateStaticClass, \
			); \
		} \
		return PrivateStaticClass; \
	}
```

+ UObjectBase.h  
```
struct FCompiledInDefer
{
	FCompiledInDefer(class UClass *(*InRegister)(), class UClass *(*InStaticClass)(), const TCHAR* PackageName, const TCHAR* Name, ...)
	{
        // Stashes the singleton function that builds a compiled in class
        // 向数组中添加InRegister
        // static TArray<class UClass *(*)()> DeferredCompiledInRegistration;
		UObjectCompiledInDefer(InRegister, InStaticClass, Name, PackageName, ...);
	}
};

// .gen.cpp
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(
    Z_Construct_UClass_UMyClass, 
    &UMyClass::StaticClass, 
    TEXT("/Script/Confuse"), 
    TEXT("UMyClass"), ...);
```

## 收集 FUNCTION
收集内置的函数  
### IMPLEMENT_CAST_FUNCTION
+ ScriptCore.cpp  
```
#define IMPLEMENT_FUNCTION(func) \
	static FNativeFunctionRegistrar UObject##func##Registar(UObject::StaticClass(),#func,&UObject::func);

#define IMPLEMENT_CAST_FUNCTION(CastIndex, func) \
	IMPLEMENT_FUNCTION(func); \
	static uint8 UObject##func##CastTemp = GRegisterCast( CastIndex, &UObject::func );

// IMPLEMENT_CAST_FUNCTION定义一些Object转换函数
DEFINE_FUNCTION(UObject::execObjectToBool)
{
	UObject* Obj=NULL;
	Stack.Step( Stack.Object, &Obj );
	*(bool*)RESULT_PARAM = Obj != NULL;
}
IMPLEMENT_CAST_FUNCTION( CST_ObjectToBool, execObjectToBool );
```

展开后  
```
void UObject::execObjectToBool( UObject* Context, FFrame& Stack, void*const Z_Param__Result )
{
	UObject* Obj=NULL;
	Stack.Step( Stack.Object, &Obj );
	*(bool*)Z_Param__Result = Obj != NULL;
}

static FNativeFunctionRegistrar UObjectexecObjectToBoolRegistar(UObject::StaticClass(),"execObjectToBool",&UObject::execObjectToBool);
static uint8 UObjectexecObjectToBoolCastTemp = GRegisterCast( 0x47, &UObject::func );
```

+ CoreNative.h
```
// 函数名到native函数的映射的结构
struct FNativeFunctionRegistrar
{
	FNativeFunctionRegistrar(class UClass* Class, const ANSICHAR* InName, FNativeFuncPtr InPointer)
	{
		RegisterFunction(Class, InName, InPointer);
	}
	static COREUOBJECT_API void RegisterFunction(class UClass* Class, const ANSICHAR* InName, FNativeFuncPtr InPointer);
}

// Class.cpp
void FNativeFunctionRegistrar::RegisterFunction(class UClass* Class, const ANSICHAR* InName, FNativeFuncPtr InPointer)
{
	Class->AddNativeFunction(InName, InPointer);
}
void UClass::AddNativeFunction(const ANSICHAR* InName, FNativeFuncPtr InPointer)
{
	// 通过static把函数收集到class的数组中
	// TArray<FNativeFunctionLookup> NativeFunctionLookupTable;
	new(NativeFunctionLookupTable) FNativeFunctionLookup(InFName,InPointer);
}
```

```
uint8 COREUOBJECT_API GRegisterCast( int32 CastCode, const FNativeFuncPtr& Func );

// ScriptCore.cpp
COREUOBJECT_API uint8 GRegisterCast( int32 CastCode, const FNativeFuncPtr& Func )
{
	GCasts[CastCode] = Func;
}

COREUOBJECT_API FNativeFuncPtr GCasts[CST_Max]; // CST_Max=255
```

总结:  
FNativeFunctionRegistrar结构将函数存入UClass::NativeFunctionLookupTable  
GRegisterCast()将函数存入全局变量GCasts[255]中

### IMPLEMENT_VM_FUNCTION
+ ScriptCore.cpp  
```
#define IMPLEMENT_VM_FUNCTION(BytecodeIndex, func) \
	STORE_INSTRUCTION_NAME(BytecodeIndex) \
	IMPLEMENT_FUNCTION(func) \
	static uint8 UObject##func##BytecodeTemp = GRegisterNative( BytecodeIndex, &UObject::func );

	// GRegisterNative 将函数指针存储到全局数组 FNativeFuncPtr GNatives[EX_Max]

const char* GNativeFuncNames[EX_Max]; // EX_Max=256

#define STORE_INSTRUCTION_NAME(inst) \
static struct F##inst##Registrar \
{ \
	F##inst##Registrar() \
	{ \
		GNativeFuncNames[inst] = #inst; \
	} \
} inst##RegistrarInst;
```

举例  
```
DEFINE_FUNCTION(UObject::execTrue)
{
	*(bool*)RESULT_PARAM = true;
}
IMPLEMENT_VM_FUNCTION( EX_True, execTrue );

void UObject::execTrue( UObject* Context, FFrame& Stack, void*const Z_Param__Result )
{
	*(bool*)Z_Param__Result = true;
}

// Object.h
DECLARE_FUNCTION(execTrue);
static void UObject::execTrue( UObject* Context, FFrame& Stack, void*const Z_Param__Result );
```

总结:  
宏定义的static结构,将函数名存入全局变量const char* GNativeFuncNames[256]  
FNativeFunctionRegistrar结构将函数存入UClass::NativeFunctionLookupTable  
GRegisterCast()将函数存入全局变量GCasts[255]中

