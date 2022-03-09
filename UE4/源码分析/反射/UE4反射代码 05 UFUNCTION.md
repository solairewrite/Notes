# UE4反射代码 05 UFUNCTION
## BlueprintNativeEvent
+ .h, .cpp  
自己声明,定义的函数  
```
UFUNCTION(BlueprintNativeEvent, Category = "Hello")
    void NativeFunc();

void UMyClass::NativeFunc_Implementation()
{
}
```

+ .generated.h  
UHT 声明 exec 函数和 _Implementation 函数  
```
virtual void NativeFunc_Implementation();

DECLARE_FUNCTION(execNativeFunc);
// 展开后
static void execNativeFunc( UObject* Context, FFrame& Stack, RESULT_DECL );
```

+ .gen.cpp  
exec 函数调用了 _Implementation 函数  
```
DEFINE_FUNCTION(UMyClass::execNativeFunc)
{
    P_FINISH;
    P_NATIVE_BEGIN;
    P_THIS->NativeFunc_Implementation();
    P_NATIVE_END;
}

// 展开后
void UMyClass::execNativeFunc( UObject* Context, FFrame& Stack, void*const Z_Param__Result )
{
    Stack.Code += !!Stack.Code; // 增加代码指针,除非为空
    {
        ((ClassType*)(Context))->NativeFunc_Implementation();
    }
}
```

自定义的 NativeFunc 函数,用 ProcessEvent 封装,在map中根据函数名查找对应的 exec 函数  
```
static FName NAME_UMyClass_NativeFunc = FName(TEXT("NativeFunc"));
void UMyClass::NativeFunc()
{
    ProcessEvent(FindFunctionChecked(NAME_UMyClass_NativeFunc),NULL);
}

// 注册函数
void UMyClass::StaticRegisterNativesUMyClass()
{
    UClass* Class = UMyClass::StaticClass();
    static const FNameNativePtrPair Funcs[] = {
        { "CallableFunc", &UMyClass::execCallableFunc },
        { "NativeFunc", &UMyClass::execNativeFunc },
    };
    FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
}
```

函数调用关系总结:  
自定义函数 NativeFunc -> exec 函数 -> _Implementation 函数  

## BlueprintCallable
+ .h, .cpp
```
UFUNCTION(BlueprintCallable, Category = "Hello")
    void CallableFunc();

void UMyClass::CallableFunc()
{
}
```

+ .generated.h  
```
static void execCallableFunc( UObject* Context, FFrame& Stack, RESULT_DECL );
```

+ .gen.cpp  
```
void UMyClass::execCallableFunc( UObject* Context, FFrame& Stack, void*const Z_Param__Result )
{
    Stack.Code += !!Stack.Code; // 增加代码指针,除非为空
    {
        ((ClassType*)(Context))->CallableFunc();
    }
}
```

函数调用关系总结:  
exec 函数 -> 自定义 CallableFunc  

## BlueprintImplementableEvent
+ .h, .cpp
```
UFUNCTION(BlueprintImplementableEvent, Category = "Hello")
    void ImplementableFunc();
```

+ .gen.cpp  
```
static FName NAME_UMyClass_ImplementableFunc = FName(TEXT("ImplementableFunc"));
void UMyClass::ImplementableFunc()
{
    ProcessEvent(FindFunctionChecked(NAME_UMyClass_ImplementableFunc),NULL);
}
```

总结:  
自定义函数 ImplementableFunc 调用了,根据函数名查找到的函数  

## 问题:
1. 函数注册流程  
StaticRegisterNativesUMyClass  
FNativeFunctionRegistrar::RegisterFunctions  
FindFunctionChecked  

2. BlueprintImplementableEvent  
根据函数名,查找到的函数在哪里定义的,在哪里注册的  
