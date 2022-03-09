# UE4反射代码 16 构造UFUNCTION
## 接口函数的定义都在接口这边
+ .h
```
UFUNCTION(BlueprintCallable, BlueprintImplementableEvent, Category = "Trigger Reaction")
    bool ReactToTrigger() const;

UFUNCTION(BlueprintNativeEvent)
    void NativeInterfaceFunc();
```

+ .generated.h
```
// 这里函数已经实现,所以.cpp中不能实现
virtual void NativeInterfaceFunc_Implementation() {}

static void execNativeInterfaceFunc( UObject* Context, FFrame& Stack, RESULT_DECL )

static void Execute_NativeInterfaceFunc(UObject* O);
static bool Execute_ReactToTrigger(const UObject* O);
```

+ .gen.cpp
```
void IReactToTriggerInterface::execNativeInterfaceFunc( UObject* Context, FFrame& Stack, RESULT_DECL )
{
    Stack.Code += !!Stack.Code
    {
        ((ThisClass*)(Context))->NativeInterfaceFunc_Implementation();
    }
}

static FName NAME_UReactToTriggerInterface_NativeInterfaceFunc = FName(TEXT("NativeInterfaceFunc"));
void IReactToTriggerInterface::Execute_NativeInterfaceFunc(UObject* O)
{
    通过名字查找函数
    UFunction* const Func = O->FindFunction(NAME_UReactToTriggerInterface_NativeInterfaceFunc);
    if (Func)
    {
        O->ProcessEvent(Func, NULL);
    }
    // 如果BP没找到函数,则调用C++中的_Implementation实现
    else if (auto I = (IReactToTriggerInterface*)(O->GetNativeInterfaceAddress(UReactToTriggerInterface::StaticClass())))
    {
        I->NativeInterfaceFunc_Implementation();
    }
}

static FName NAME_UReactToTriggerInterface_ReactToTrigger = FName(TEXT("ReactToTrigger"));
bool IReactToTriggerInterface::Execute_ReactToTrigger(const UObject* O)
{
    ReactToTriggerInterface_eventReactToTrigger_Parms Parms;
    // 通过名字查找函数
    UFunction* const Func = O->FindFunction(NAME_UReactToTriggerInterface_ReactToTrigger);
    if (Func)
    {
        const_cast<UObject*>(O)->ProcessEvent(Func, &Parms);
    }
    return Parms.ReturnValue;
}
```

## 实现接口的类
```
UFunction* Z_Construct_UFunction_UMyClass_AddHP()
{
    static UFunction* ReturnFunction = nullptr;
    if (!ReturnFunction)
    {
        UE4CodeGen_Private::ConstructUFunction(ReturnFunction, ...);
    }
    return ReturnFunction;
}
```

+ Class.cpp
```
void UClass::CreateLinkAndAddChildFunctionsToMap(const FClassFunctionLinkInfo* Functions, uint32 NumFunctions)
{
	for (; NumFunctions; --NumFunctions, ++Functions)
	{
        // 这里Functions->CreateFuncPtr 是 Z_Construct_UFunction_UMyClass_AddHP
		UFunction*  Func = Functions->CreateFuncPtr();
	}
}
```

+ UObjectGlobals.cpp
```
void ConstructUFunction(UFunction*& OutFunction, const FFunctionParams& Params)
{
    UObject*   (*OuterFunc)() = Params.OuterFunc;
    UFunction* (*SuperFunc)() = Params.SuperFunc;

    UObject*   Outer = OuterFunc ? OuterFunc() : nullptr;
    UFunction* Super = SuperFunc ? SuperFunc() : nullptr;

    if (OutFunction)
    {
        return;
    }

    UFunction* NewFunction;
    if (Params.FunctionFlags & FUNC_Delegate) // 生成代理函数
    {
        if (Params.OwningClassName == nullptr)
        {
            NewFunction = new (...) UDelegateFunction(...);
        }
        else
        {
            USparseDelegateFunction* NewSparseFunction = new (...) USparseDelegateFunction(...);
            NewSparseFunction->OwningClassName = FName(Params.OwningClassName);
            NewSparseFunction->DelegateName = FName(Params.DelegateName);
            NewFunction = NewSparseFunction;
        }
    }
    else
    {
        NewFunction = new (EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags) UFunction(
            FObjectInitializer(),
            Super,
            Params.FunctionFlags,
            Params.StructureSize
        );
    }
    OutFunction = NewFunction;

    // 生成属性列表
	ConstructFProperties(NewFunction, Params.PropertyArray, Params.NumProperties);

    // 函数绑定
	NewFunction->Bind();
    // 函数链接
	NewFunction->StaticLink();
	}
```

## 绑定
即设置函数指针 FNativeFuncPtr UFunction::Func  
```
typedef void (*FNativeFuncPtr)(UObject* Context, FFrame& TheStack, RESULT_DECL);

FNativeFuncPtr UFunction::Func;

void UFunction::Bind()
{
	UClass* OwnerClass = GetOwnerClass();

	// 非native函数指向蓝图调用
	if (!HasAnyFunctionFlags(FUNC_Native))
	{
		Func = &UObject::ProcessInternal;
	}
	else
	{
		// 在之前注册的NativeFunctionLookupTable里面查找函数
		FName Name = GetFName();
		FNativeFunctionLookup* Found = OwnerClass->NativeFunctionLookupTable.FindByPredicate(
            // lambda表达式
            // [=]: 表达式内可以按值使用外面的变量
            // (): 函数参数
            // {}: 函数体
            [=](const FNativeFunctionLookup& NativeFunctionLookup){ return Name == NativeFunctionLookup.Name; }
            );
		if (Found)
		{
			Func = Found->Pointer;
		}
	}
}

void UClass::Bind()
{
	UStruct::Bind();

	UClass* SuperClass = GetSuperClass();
	if (SuperClass && (ClassConstructor == nullptr || ClassAddReferencedObjects == nullptr
		|| ClassVTableHelperCtorCaller == nullptr
		))
	{
		// 确保基类已经绑定
		SuperClass->Bind();
        // 绑定构造函数指针
		if (!ClassConstructor)
		{
			ClassConstructor = SuperClass->ClassConstructor;
		}
		if (!ClassVTableHelperCtorCaller) // 热加载函数指针
		{
			ClassVTableHelperCtorCaller = SuperClass->ClassVTableHelperCtorCaller;
		}
		if (!ClassAddReferencedObjects) // ARO函数指针
		{
			ClassAddReferencedObjects = SuperClass->ClassAddReferencedObjects;
		}
        // 这3个函数和GetPrivateStaticClassBody里传入的相同
	}
}
```