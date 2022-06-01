# GameFeatures_02_加载GameFeature
## 目录
- [GameFeatures_02_加载GameFeature](#gamefeatures_02_加载gamefeature)
    - [目录](#目录)
    - [参考](#参考)
    - [概述](#概述)
    - [创建GameFeature](#创建gamefeature)
    - [初始化](#初始化)
        - [UGameFeaturesSubsystem::Initialize](#ugamefeaturessubsysteminitialize)
        - [UGameFeaturesProjectPolicies 控制加载哪些GF](#ugamefeaturesprojectpolicies-控制加载哪些gf)
        - [UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugin 加载GF](#ugamefeaturessubsystemloadbuiltingamefeatureplugin-加载gf)

## 参考
知乎@大钊 [《InsideUE5》GameFeatures架构（二）基础用法](https://zhuanlan.zhihu.com/p/470184973)  
知乎@大钊 [《InsideUE5》GameFeatures架构（三）初始化](https://zhuanlan.zhihu.com/p/473535854)  

## 概述
等学习Lyra项目一段时间后再回到这里,那时候就知道GF是干甚么的了  

创建自定义的GF后,引擎会自动创建一个同名的GameFeatureData  
`TArray<UGameFeatureAction*> UGameFeatureData::Actions`描述了这个GF要执行的动作列表,由我们自己配置有哪些Action  
GF在激活的时候,会执行`UGameFeatureAction::OnGameFeatureActivating`  
GF反激活时,会执行`UGameFeatureAction::OnGameFeatureDeactivating`  
GF的主要逻辑就在这两个函数里面  

## 创建GameFeature
参考链接中的文章  

## 初始化
### UGameFeaturesSubsystem::Initialize
创建GF加载策略,注册AssetManager创建完成的回调  

```
void UGameFeaturesSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    // LyraGameFeaturePolicy
    const FSoftClassPath& PolicyClassPath = GetDefault<UGameFeaturesSubsystemSettings>()->GameFeaturesManagerClassName;
    
    UClass* SingletonClass = LoadClass<UGameFeaturesProjectPolicies>(nullptr, *PolicyClassPath.ToString());

    GameSpecificPolicies = NewObject<UGameFeaturesProjectPolicies>(this, SingletonClass);

    UAssetManager::CallOrRegister_OnAssetManagerCreated(FSimpleMulticastDelegate::FDelegate::CreateUObject(this, &ThisClass::OnAssetManagerCreated));
}

void UGameFeaturesSubsystem::OnAssetManagerCreated()
{
    GameSpecificPolicies->InitGameFeatureManager();
}
```

### UGameFeaturesProjectPolicies 控制加载哪些GF
在Project Settings -> Game Features中设置自己的加载策略  

```
void UDefaultGameFeaturesProjectPolicies::InitGameFeatureManager()
{
    // 自定义GF加载过滤器
    // FGameFeaturePluginDetails读取.uplugin文件中的字段
    // 需要配置TArray<FString> UGameFeaturesSubsystemSettings::AdditionalPluginMetadataKeys 来定义读取哪些字段
    auto AdditionalFilter = [&](const FString& PluginFilename, const FGameFeaturePluginDetails& PluginDetails, FBuiltInGameFeaturePluginBehaviorOptions& OutOptions) -> bool
	{
        // 默认全部加载,可以在这里自定义加载哪些GF
		return true;
	};

    // 加载GF
	UGameFeaturesSubsystem::Get().LoadBuiltInGameFeaturePlugins(AdditionalFilter);
}
```

### UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugin 加载GF
遍历当前项目Plugins目录下所有插件,根据加载策略加载GF  

```
void UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugins(FBuiltInPluginAdditionalFilters AdditionalFilter)
{
    TArray<TSharedRef<IPlugin>> EnabledPlugins = IPluginManager::Get().GetEnabledPlugins();
    for (const TSharedRef<IPlugin>& Plugin : EnabledPlugins)
	{
		LoadBuiltInGameFeaturePlugin(Plugin, AdditionalFilter);
	}
}

void UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugin(const TSharedRef<IPlugin>& Plugin, FBuiltInPluginAdditionalFilters AdditionalFilter)
{
    // ...Lyra\Plugins\GameFeatures\ShooterCore\ShooterCore.uplugin
    const FString& PluginDescriptorFilename = Plugin->GetDescriptorFileName();

    // GF必须位于: 各种项目目录/Plugins/GameFeatures,这里省略文件路径检测代码

    const FString PluginURL = GetPluginURL_FileProtocol(PluginDescriptorFilename);
    // 判断GF是否可加载,这里未读取.uplugin文件,性能快
    if (GameSpecificPolicies->IsPluginAllowed(PluginURL))
    {
        // 读取.uplugin文件内的字段
        FGameFeaturePluginDetails PluginDetails;
        if (GetGameFeaturePluginDetails(PluginDescriptorFilename, PluginDetails))
        {
            bool bShouldProcess = AdditionalFilter(PluginDescriptorFilename, PluginDetails, BehaviorOptions);
            if (bShouldProcess)
            {
                // 创建状态机
                UGameFeaturePluginStateMachine* StateMachine = FindOrCreateGameFeaturePluginStateMachine(PluginURL);

                // ShooterCore.uplugin, "BuiltInInitialFeatureState": "Registered",
                // 对于Lyra项目,DestinationState 是 Registered
                if (StateMachine->GetCurrentState() >= DestinationState)
                {
                    LoadGameFeaturePluginComplete(StateMachine, MakeValue());
                }
                else
                {
                    // 设置目标状态后,状态机会一个一个的切换下一个状态,直到到达目标状态
                    StateMachine->SetDestinationState(DestinationState, FGameFeatureStateTransitionComplete::CreateUObject(this, &ThisClass::LoadGameFeaturePluginComplete));
                }
            }
        }
    }
}
```
