# UE4反射代码 11 构造UEnum
## UEnum
+ UObjectBase.cpp
```
static void UObjectLoadAllCompiledInStructs()
{
    // static FCompiledInDeferEnum中注册
    TArray<FPendingEnumRegistrant> PendingEnumRegistrants = MoveTemp(GetDeferredCompiledInEnumRegistration());

    for (const FPendingEnumRegistrant& EnumRegistrant : PendingEnumRegistrants)
	{
        // 这里RegisterFn是EMyEnum_StaticEnum,创建UEnum*
		EnumRegistrant.RegisterFn();
	}
}
```

+ MyEnum.gen.cpp
```
// EMyEnum_StaticEnum调用Z_Construct_UEnum_Confuse_EMyEnum
UEnum* Z_Construct_UEnum_Confuse_EMyEnum()
{
    static UEnum* ReturnEnum = nullptr;
    if (!ReturnEnum)
    {
        // 创建UEnum*单例单例
        // 设置 TArray<TPair<FName, int64>> UEnum::Names;
        UE4CodeGen_Private::ConstructUEnum(ReturnEnum, EnumParams);
    }
    return ReturnEnum;
}
```


+ UObjectGlobals.cpp
```
	void ConstructUEnum(UEnum*& OutEnum, const FEnumParams& Params)
	{
		UObject* (*OuterFunc)() = Params.OuterFunc; // 获取返回Package的函数指针,定义在Confuse.init.gen.cpp

		UObject* Outer = OuterFunc ? OuterFunc() : nullptr; // 获取Package

		if (OutEnum)
		{
			return;
		}
		// DECLARE_CLASS中声明的operator new()调用StaticAllocateObject分配内存,然后再构造
		UEnum* NewEnum = new (EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags) UEnum(FObjectInitializer());
		OutEnum = NewEnum;
		// 将枚举名-枚举值储存到数组中
		TArray<TPair<FName, int64>> EnumNames;
		EnumNames.Reserve(Params.NumEnumerators);
		for (const FEnumeratorParam* Enumerator = Params.EnumeratorParams, *EnumeratorEnd = Enumerator + Params.NumEnumerators; Enumerator != EnumeratorEnd; ++Enumerator)
		{
			EnumNames.Emplace(UTF8_TO_TCHAR(Enumerator->NameUTF8), Enumerator->Value);
		}

        // 设置枚举项数组
		NewEnum->SetEnums(EnumNames, (UEnum::ECppForm)Params.CppForm, Params.EnumFlags, Params.DynamicType == EDynamicType::NotDynamic);
		NewEnum->CppType = UTF8_TO_TCHAR(Params.CppTypeUTF8);

		if (Params.DisplayNameFunc)
		{
            // 设置显示名字的函数
			NewEnum->SetEnumDisplayNameFn(Params.DisplayNameFunc);
		}
	}
```


+ Enum.cpp
```
// 全局UEnum Map
TMap<FName, UEnum*> UEnum::AllEnumNames;

bool UEnum::SetEnums(TArray<TPair<FName, int64>>& InNames, UEnum::ECppForm InCppForm, EEnumFlags InFlags, bool bAddMaxKeyIfMissing)
{
	if (Names.Num() > 0)
	{
		RemoveNamesFromMasterList(); // 从全局map中删除原来的名字
	}
	// 设置TArray<TPair<FName, int64>> UEnum::Names;
	Names     = InNames;
	CppForm   = InCppForm;
	EnumFlags = InFlags;

	if (bAddMaxKeyIfMissing)
	{
		if (!ContainsExistingMax()) // 确保枚举最后一个是#_MAX
		{
			FName MaxEnumItem = *GenerateFullEnumName(*(GenerateEnumPrefix() + TEXT("_MAX")));
			if (LookupEnumName(MaxEnumItem) != INDEX_NONE)
			{
				return false;
			}

			Names.Emplace(MaxEnumItem, GetMaxEnumValue() + 1);
		}
	}
	AddNamesToMasterList();

	return true;
}
```