# UE4反射代码 13 构造UClass
+ MyClass.gen.cpp
```
struct Z_Construct_UClass_UMyClass_Statics
{
    // DependentSingletons 是 UObject*(*)()函数指针的const数组
    static UObject* (*const DependentSingletons[])();

    static const UE4CodeGen_Private::FImplementedInterfaceParams InterfaceParams[];
}

UObject* (*const Z_Construct_UClass_UMyClass_Statics::DependentSingletons[])() = {
    (UObject* (*)())Z_Construct_UClass_UObject, // 依赖基类UObject
    (UObject* (*)())Z_Construct_UPackage__Script_Confuse, // 依赖Package
};

const UE4CodeGen_Private::FImplementedInterfaceParams Z_Construct_UClass_UMyClass_Statics::InterfaceParams[] = 
{
    { 
        Z_Construct_UClass_UReactToTriggerInterface_NoRegister, // 构造Interface所属的UClass*
        (int32)VTABLE_OFFSET(UMyClass, IReactToTriggerInterface), // 多重继承的指针偏移
        false // 是否在蓝图实现
    },
};
```


+ UObjectBase.cpp
```
static void UObjectLoadAllCompiledInDefaultProperties()
{
    // static注册的FCompiledInDefer
    // Registrant 是 Z_Construct_UClass_UMyClass
    for (UClass* (*Registrant)() : PendingRegistrants)
    {
        // 里面会调用 UE4CodeGen_Private::ConstructUClass
        UClass* Class = Registrant();
    }
}
```

```
void ConstructUClass(UClass*& OutClass, const FClassParams& Params)
{
    // 构造依赖的对象,基类和Package
    for (UObject* (*const *SingletonFunc)() = Params.DependencySingletonFuncArray, 
        *(*const *SingletonFuncEnd)() = SingletonFunc + Params.NumDependencySingletons; 
        SingletonFunc != SingletonFuncEnd; 
        ++SingletonFunc)
    {
        (*SingletonFunc)();
    }

    // 获取已经生成的UClass*
    UClass* NewClass = Params.ClassNoRegisterFunc();
    OutClass = NewClass;

    // 确保UClass*已经注册
    UObjectForceRegistration(NewClass);

	// 构造所有函数
    // 添加函数到 TMap<FName, UFunction*> UClass::FuncMap;
    // 添加函数到链表头 UField* UStruct::Children;
	NewClass->CreateLinkAndAddChildFunctionsToMap(Params.FunctionLinkArray, Params.NumFunctions);

    // 构造所有属性
    // 将属性添加到子属性链表头 FField* UStruct::ChildProperties;
	ConstructFProperties(NewClass, Params.PropertyArray, Params.NumProperties);

    // ini配置文件名
    // FName UClass::ClassConfigName;
    if (Params.ClassConfigNameUTF8)
    {
        NewClass->ClassConfigName = FName(UTF8_TO_TCHAR(Params.ClassConfigNameUTF8));
    }

    // 构造所有实现的接口
    if (int32 NumImplementedInterfaces = Params.NumImplementedInterfaces)
    {
        NewClass->Interfaces.Reserve(NumImplementedInterfaces);
        for (const FImplementedInterfaceParams* ImplementedInterface = Params.ImplementedInterfaceArray, 
            *ImplementedInterfaceEnd = ImplementedInterface + NumImplementedInterfaces; 
            ImplementedInterface != ImplementedInterfaceEnd; 
            ++ImplementedInterface)
        {
            UClass* (*ClassFunc)() = ImplementedInterface->ClassFunc;
            UClass* InterfaceClass = ClassFunc ? ClassFunc() : nullptr; // 获取Interface所属的UClass*对象

            // 添加实现的接口
            // TArray<FImplementedInterface> UClass::Interfaces;
            NewClass->Interfaces.Emplace(InterfaceClass, ImplementedInterface->Offset, ImplementedInterface->bImplementedByK2);
        }
    }

    // 链接
    NewClass->StaticLink();
}
```

+ Class.cpp
```
void UClass::CreateLinkAndAddChildFunctionsToMap(const FClassFunctionLinkInfo* Functions, uint32 NumFunctions)
{
	for (; NumFunctions; --NumFunctions, ++Functions)
	{
		const char* FuncNameUTF8 = Functions->FuncNameUTF8;
		UFunction*  Func         = Functions->CreateFuncPtr(); // 构造UFunction*对象

        // 函数放在链表头
        // UField* UStruct::Children;
		Func->Next = Children;
		Children = Func;

        // 添加函数到 TMap<FName, UFunction*> UClass::FuncMap;
		AddFunctionToFunctionMap(Func, FName(UTF8_TO_TCHAR(FuncNameUTF8)));
	}
}
```