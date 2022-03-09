# UE4反射代码 15 构造FProperty
+ MyClass.gen.cpp
```
	struct Z_Construct_UClass_UMyClass_Statics
	{
		static const UE4CodeGen_Private::FFloatPropertyParams NewProp_Score;
		static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[];
	};

    const UE4CodeGen_Private::FFloatPropertyParams Z_Construct_UClass_UMyClass_Statics::NewProp_Score = 
    { 
        "Score", 
        nullptr, // const char* RepNotifyFuncUTF8,RepNotify函数名称
        (EPropertyFlags)0x0010000000000004, 
        UE4CodeGen_Private::EPropertyGenFlags::Float, 
        RF_Public|RF_Transient|RF_MarkAsNative, 
        1, // int32 ArrayDim; 数组维度
        STRUCT_OFFSET(UMyClass, Score), // 属性的结构偏移地址
    };

    const UE4CodeGen_Private::FPropertyParamsBase* const Z_Construct_UClass_UMyClass_Statics::PropPointers[] = 
    {
		(const UE4CodeGen_Private::FPropertyParamsBase*)&Z_Construct_UClass_UMyClass_Statics::NewProp_Score,
	};
```

+ UObjectGlobals.cpp  
`ConstructUScriptStruct`, `ConstructUClass`里面会调用`ConstructFProperties`,参数是上面的PropPointers  
```
void ConstructFProperties(UObject* Outer, const FPropertyParamsBase* const* PropertyArray, int32 NumProperties)
{
    // 将指针移到最后,倒序遍历
    PropertyArray += NumProperties;
    while (NumProperties)
    {
        // 将属性添加到子属性链表头 FField* UStruct::ChildProperties;
        ConstructFProperty(Outer, PropertyArray, NumProperties);
    }
}

void ConstructFProperty(FFieldVariant Outer, const FPropertyParamsBase* const*& PropertyArray, int32& NumProperties)
{
    const FPropertyParamsBase* PropBase = *--PropertyArray;

    uint32 ReadMore = 0; // 需要几个子属性

    FProperty* NewProp = nullptr;
    switch (PropBase->Flags & PropertyTypeMask)
    {
        // 这里包含了所有类型的属性

        case EPropertyGenFlags::Float:
        {
            const FFloatPropertyParams* Prop = (const FFloatPropertyParams*)PropBase;
            // 构造FProperty对象
            // 构造函数中会将FProperty自身加入Outer中
            NewProp = new FFloatProperty(Outer, UTF8_TO_TCHAR(Prop->NameUTF8), Prop->ObjectFlags, Prop->Offset, Prop->PropertyFlags);
        }
        break;

        case EPropertyGenFlags::Array:
        {
            const FArrayPropertyParams* Prop = (const FArrayPropertyParams*)PropBase;
            NewProp = new FArrayProperty(Outer, UTF8_TO_TCHAR(Prop->NameUTF8), Prop->ObjectFlags, Prop->Offset, Prop->PropertyFlags, Prop->ArrayFlags);

            // 需要一个子属性
            ReadMore = 1;
        }
        break;

        // 设置属性维度,单属性为1,int32 prop[10]为10
        NewProp->ArrayDim = PropBase->ArrayDim;
		if (PropBase->RepNotifyFuncUTF8)
		{
			NewProp->RepNotifyFunc = FName(UTF8_TO_TCHAR(PropBase->RepNotifyFuncUTF8));
		}

		--NumProperties;

		for (; ReadMore; --ReadMore)
		{
            // 构造子属性,这里Outer是NewProp
			ConstructFProperty(NewProp, PropertyArray, NumProperties);
		}
	}
```

```
// Property.cpp
FProperty::FProperty(...)
{
    Init();
}

void FProperty::Init()
{
	if (GetOwner<UObject>())
	{
		UField* OwnerField = GetOwnerChecked<UField>();
		OwnerField->AddCppProperty(this); // 虚函数
	}
}

// Class.cpp
// 指向子属性链表头
// FField* UStruct::ChildProperties;
void UStruct::AddCppProperty(FProperty* Property)
{
	Property->Next = ChildProperties;
	ChildProperties = Property;
}
```