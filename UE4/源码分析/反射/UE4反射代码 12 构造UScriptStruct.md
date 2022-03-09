# UE4反射代码 12 构造UScriptStruct
USTRUCT不能包含UFUNCTION  

+ UObjectGlobals.cpp
```
	void ConstructUScriptStruct(UScriptStruct*& OutStruct, const FStructParams& Params)
	{
		UObject*                      (*OuterFunc)()     = Params.OuterFunc;
		UScriptStruct*                (*SuperFunc)()     = Params.SuperFunc;
		UScriptStruct::ICppStructOps* (*StructOpsFunc)() = (UScriptStruct::ICppStructOps* (*)())Params.StructOpsFunc;

		UObject*                      Outer     = OuterFunc     ? OuterFunc() : nullptr; // 构造Outer
		UScriptStruct*                Super     = SuperFunc     ? SuperFunc() : nullptr; // 构造SuperStruct
		UScriptStruct::ICppStructOps* StructOps = StructOpsFunc ? StructOpsFunc() : nullptr; // 构造结构操作类

		if (OutStruct)
		{
			return;
		}

        // 构造UScriptStruct
		UScriptStruct* NewStruct = new(EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags) 
            UScriptStruct(FObjectInitializer(), Super, StructOps, (EStructFlags)Params.StructFlags, Params.SizeOf, Params.AlignOf);
		OutStruct = NewStruct;

        // 构造属性集合
		// 将属性添加到子属性链表头 FField* UStruct::ChildProperties;
		ConstructFProperties(NewStruct, Params.PropertyArray, Params.NumProperties);

        // 链接
		NewStruct->StaticLink();
	}
```