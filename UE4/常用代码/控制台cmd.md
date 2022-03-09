# 控制台cmd
声明控制台变量有两种方式:  
1. 直接注册控制台变量  
2. 注册一个对现有变量的引用  
## 直接注册控制台变量
UE4Mix.h 声明全局控制台变量  
所有的.cpp文件都可以使用  
```
static TAutoConsoleVariable<int32> CVARDebugLevel(
	TEXT("DebugLevel"),
	0,
	TEXT("调试等级,控制是否显示调试球等,数值越大,能显示的越多"),
	ECVF_Cheat
);
```
PlayerCharacter_Knight.cpp 引用控制台变量
```
#include "UE4Mix.h"

// 这行代码会引发警告,注释掉后也能编译通过
//extern TAutoConsoleVariable<int32> CVARDebugLevel;

if (CVARDebugLevel.GetValueOnGameThread() > 0)
{}
```
## 注册一个对现有变量的引用  
只能在当前.cpp文件使用  
```
// 声明控制台变量
static int32 DebugHealth = 0;
FAutoConsoleVariableRef CVARDebugHealth(
	TEXT("Coop.DebugHealth"), // 控制台命令字符
	DebugHealth,
	TEXT("Show Debug String of Health Component"),
	ECVF_Cheat
);

// 控制台变量决定是否显示log
if (DebugHealth >= 1)
{
    UE_LOG(LogTemp, Warning, TEXT("血量改变: %s"), *FString::SanitizeFloat(Health));
}
```