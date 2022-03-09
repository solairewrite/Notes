# LusState
```
// 加载lua文件,返回文件内容的缓存
// fn是简短的文件名,返回完整的文件路径filepath
typedef TArray<uint8> (*LoadFileDelegate) (const char* fn, FString& filepath);

// 加载文件并执行,加载方式依赖于加载代理,查看setLoadFileDelegate
LuaVar doFile(const char* fn, LuaVar* pEnv = nullptr);

// 通过key调用函数
template<typename ...ARGS>
LuaVar call(const char* key,ARGS&& ...args) {
    LuaVar f = get(key);
    return f.call(std::forward<ARGS>(args)...);
}

// 设置加载lua代码的代理函数
void setLoadFileDelegate(LoadFileDelegate func);

LoadFileDelegate loadFileDelegate;

TArray<uint8> loadFile(const char* fn,FString& filepath);

lua_State* L;

// 缓存函数/属性
struct ClassCache {
    typedef TMap<FString, TWeakObjectPtr<UFunction>> CacheFuncItem;
    typedef TMap<TWeakObjectPtr<UClass>, CacheFuncItem> CacheFuncMap;

    typedef TMap<FString, TWeakFieldPtr<FProperty>> CachePropItem;
    typedef TMap<TWeakObjectPtr<UClass>, CachePropItem> CachePropMap;

    CacheFuncMap cacheFuncMap;
    CachePropMap cachePropMap;
} classMap;

// struct GenericUserData 里面维护了 void* ud
typedef TMap<UObject*, GenericUserData*> UObjectRefMap;
// 维护push到lua里面的UObject
UObjectRefMap objRefs;

// 维护UGameInstance指针用来搜索LuaState
UGameInstance* pGI;
```
