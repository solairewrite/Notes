# UnLua_02_Object绑定Lua
## 目录
- [UnLua_02_Object绑定Lua](#unlua_02_object绑定lua)
    - [目录](#目录)
    - ['FLuaContext'](#fluacontext)
    - [开启引擎时LuaContext监听Object的创建](#开启引擎时luacontext监听object的创建)
        - [`UObject`的创建](#uobject的创建)
        - [LuaContext监听UObject的创建](#luacontext监听uobject的创建)
    - [`FLuaContext::TryToBindLua`在`UObject`创建时尝试为其绑定Lua](#fluacontexttrytobindlua在uobject创建时尝试为其绑定lua)

## 'FLuaContext'
继承了`FUObjectCreateListener`,可以监听UObject的创建,并尝试为其绑定Lua  

```
class FLuaContext : public FUObjectArray::FUObjectCreateListener, 
    public FUObjectArray::FUObjectDeleteListener
{
    // 当Object创建时会被调用
    virtual void NotifyUObjectCreated(const class UObjectBase *InObject, int32 Index) override;
    virtual void NotifyUObjectDeleted(const class UObjectBase *InObject, int32 Index) override;
}

void FLuaContext::NotifyUObjectCreated(const UObjectBase* InObject, int32 Index)
{
    UObject* Object = (UObject*)InObject;
    TryToBindLua(Object);
}
```

## 开启引擎时LuaContext监听Object的创建
### `UObject`的创建
`FUObjectArray GUObjectArray;`是全局单例,在创建一个Object时会发出通知  
即遍历它的属性`TArray<FUObjectCreateListener* > UObjectCreateListeners;`调用`NotifyUObjectCreated`  

Object在构造函数中会通知监听者(其中就有LuaContext)它被创建  

```
UObjectBase::UObjectBase(...)
{
    GUObjectArray.AllocateUObjectIndex(this);
}

void FUObjectArray::AllocateUObjectIndex(UObjectBase* Object)
{
    for (int32 ListenerIndex = 0; ListenerIndex < UObjectCreateListeners.Num(); ListenerIndex++)
	{
		UObjectCreateListeners[ListenerIndex]->NotifyUObjectCreated(Object,Index);
	}
}
```

### LuaContext监听UObject的创建
```
class FUnLuaModule : public IModuleInterface
{
    // 模块dll被加载后调用(引擎启动时调用)
    virtual void StartupModule() override
    {
        // 创建单例
        FLuaContext::Create();
        
        // 监听Object创建,删除
        GUObjectArray.AddUObjectCreateListener(this);
        GUObjectArray.AddUObjectDeleteListener(this);
    }
}
```

## `FLuaContext::TryToBindLua`在`UObject`创建时尝试为其绑定Lua
每个UObject都会在其创建时尝试绑定lua,但只有静态绑定或动态绑定的Object才能绑定成功  

1. 静态绑定  
    实现`IUnLuaInterface`接口,重写`GetModuleName`,获取lua文件相对于Content/Script的路径  

2. 动态绑定  
    TODO  

```
bool FLuaContext::TryToBindLua(UObject* Object)
{
    UClass* Class = Object->GetClass();

    static UClass* InterfaceClass = UUnLuaInterface::StaticClass();

    // 没实现UnLua接口
    if (!Class->ImplementsInterface(InterfaceClass))
    {
        // 动态绑定
        if (!GLuaDynamicBinding.IsValid(Class))
            return false;

        return Manager->Bind(Object, Class, *GLuaDynamicBinding.ModuleName, GLuaDynamicBinding.InitializerTableRef);
    }

    UFunction* Func = Class->FindFunctionByName(FName("GetModuleName"));

    FString ModuleName;
    UObject* CDO = Class->GetDefaultObject();
    // ModuleName: Tutorials.02_OverrideBlueprints,应该是lua文件路径
    CDO->ProcessEvent(Func, &ModuleName);

    return Manager->Bind(Object, Class, *ModuleName, GLuaDynamicBinding.InitializerTableRef);
}
```