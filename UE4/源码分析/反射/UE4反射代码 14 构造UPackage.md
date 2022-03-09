# UE4反射代码 14 构造UPackage
+ Confuse.init.gen.cpp
```
UPackage* Z_Construct_UPackage__Script_Confuse()
{
    static UPackage* ReturnPackage = nullptr;
    if (!ReturnPackage)
    {
        static const UE4CodeGen_Private::FPackageParams PackageParams = {
            "/Script/Confuse",
            ...
        };
        UE4CodeGen_Private::ConstructUPackage(ReturnPackage, PackageParams);
    }
    return ReturnPackage;
}
```

+ UObjectGlobals.cpp
```
void ConstructUPackage(UPackage*& OutPackage, const FPackageParams& Params)
{
    if (OutPackage)
    {
        return;
    }

    // 在UObjectBase::DeferredRegister时已经创建出来,现在只需查找
    UObject* FoundPackage = StaticFindObjectFast(UPackage::StaticClass(), nullptr, FName(UTF8_TO_TCHAR(Params.NameUTF8)), false, false);

    UPackage* NewPackage = CastChecked<UPackage>(FoundPackage);
    OutPackage = NewPackage;

    for (UObject* (*const *SingletonFunc)() = Params.SingletonFuncArray, *(*const *SingletonFuncEnd)() = SingletonFunc + Params.NumSingletons; SingletonFunc != SingletonFuncEnd; ++SingletonFunc)
    {
        (*SingletonFunc)(); // 构造依赖对象,这里是nullptr
    }
}
```

在InitUObject里面创建包  
```
// UObjectBase.cpp
void UObjectBase::DeferredRegister(UClass *UClassStaticClass,const TCHAR* PackageName,const TCHAR* InName)
{
    UPackage* Package = CreatePackage(PackageName);
}

// UObjectGlobals.cpp
UPackage* CreatePackage(const TCHAR* PackageName )
{
    Result = NewObject<UPackage>(nullptr, NewPackageName, RF_Public);
}
```