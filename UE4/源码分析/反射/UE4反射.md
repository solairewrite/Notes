# UE4反射
UE4使用反射可以实现序列化,编辑器的细节面板,垃圾回收,网络同步,蓝图/C++通信等功能  

反射系统是选择加入的,只有主动标记的类型,属性,方法会被反射系统追踪  
UnrealHeaderTool会收集这些信息,生成用于支持反射机制的C++代码,然后再编译工程  

## 反射系统实现原理
UObject是反射系统的核心,每一个继承UObject且支持反射系统的类型都有一个相应的UClass,或者他的子类  
UClass中包含了该类的描述信息  

### 标记
为了标记一个包含反射类型的头文件,需要添加一个特殊的include,这时UHT知道需要处理这个文件
```
#include "FileName.generated.h"
```
可以使用UClass(), UENUM(), USTRUCT(), UFUNCTION(), UPROPERTY()来标记不同的类型和成员变量  

### UnrealHeaderTool 和 UnrealBuildTool
反射代码由UHT和UBT产生  
UBT通过扫描头文件,记录所有包含反射类型的modules,当其中有头文件改变时,就会用UHT更新反射数据
UHT解析头文件,扫描标记,生成用于支持反射的代码  
举个例子,对于一个文件filename.h,反射代码包括两个部分:  
filename.generated.h和filename.gen.cpp,前者是在filename.h中必须#include的头文件  

.generated.h中生成的函数包含了XXX_Implementation之类的补全,也包含了用于蓝图中调用C++函数的转换函数,并通过GENERATED_BODY()安插到我们编写的类中  

.gen.cpp将RPC函数包装了一层,函数名在UClass中组织为Map的形式
