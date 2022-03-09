## UObject
UObject为对象系统的基类,类层次为  
```
UObjectBase
    UObjectBaseUtility
        UObject
```
### ObjectBase  
存放对象的名字 `FName NamePrivate`  
标记信息 `EObjectFlags ObjectFlags`  
类型信息 `UClass* ClassPrivate`  
Outer对象 `UObject* OuterPrivate`  
对象管理数组中的索引 `int32 InternalIndex`  

对象标记`EObjectFlags ObjectFlags`很重要,它被引擎用于表示对象加载,保存,编辑,垃圾回收时使用  

### UObjectBaseUtility  
提供一些辅助函数: Flag设置和查询,Class查询,名字查询,Linker(uasset加载器)信息  

### UObject  
提供如下功能:  
+ 创建子对象SubObject  
+ 对象Destroy相关对象处理  
+ 对象编辑  
+ 序列化  
+ 执行脚本  
+ 从config读取或保存成员变量  

## UClass
C++语言没有完整的反射功能,需要定义一个数据结构`UClass`来描述C++中的类信息,这个数据结构也称为类的元数据  
Uclass实例不仅用于描述C++(Native)类,也用来描述Blueprint生成的类  
```
UObject
    UField // class.h反射数据对象基类
        UEnum
        UProperty // UnrealTypes.h
            UNumericProperty
            UBoolProperty
            UObjectPropertyBase
            UStructProperty
            ...
        UStruct // class.h
            UClass
            UFunction
            UScriptStruct
```

### UClass
描述了C++类和脚本类的反射信息  

### StaticClass()
ObjectMacros.h  
```
#define DECLARE_CLASS
...
inline static UClass* StaticClass()
```

UHT会扫描C++.h文件,然后生成胶水层代码  
一般每个模块生成的代码放在一个文件中  
