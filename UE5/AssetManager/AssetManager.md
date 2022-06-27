# AssetManager
## 目录
- [AssetManager](#assetmanager)
  - [目录](#目录)
  - [创建AssetManager](#创建assetmanager)
    - [UEngine::Init()](#uengineinit)
    - [UGameFeaturesSubsystem::Initialize() 在创建AssetManger之前绑定静态代理](#ugamefeaturessubsysteminitialize-在创建assetmanger之前绑定静态代理)
    - [UEngine::InitializeObjectReferences() 创建并初始化AssetManager](#uengineinitializeobjectreferences-创建并初始化assetmanager)
    - [UAssetManager::StartInitialLoading() 扫描配置的PrimaryAsset,广播代理](#uassetmanagerstartinitialloading-扫描配置的primaryasset广播代理)
  - [TODO](#todo)

## 创建AssetManager
### UEngine::Init()
```
void UEngine::Init(IEngineLoop* InEngineLoop)
{
    // 遍历UEngineSubsystem,调用 USubsystem::Initialize()
    EngineSubsystemCollection->Initialize(this);

    // 创建并初始化AssetManager
    InitializeObjectReferences();
}
```

### UGameFeaturesSubsystem::Initialize() 在创建AssetManger之前绑定静态代理
```
UGameFeaturesSubsystem : UEngineSubsystem : UDynamicSubsystem : USubsystem : UObject

void UGameFeaturesSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    // 在创建AssetManager之前,绑定静态代理OnAssetManagerCreatedDelegate
    // 创建AssetManager后会立刻广播此代理
    UAssetManager::CallOrRegister_OnAssetManagerCreated(
        FSimpleMulticastDelegate::FDelegate::CreateUObject(
            this, 
            &ThisClass::OnAssetManagerCreated
        )
    );
}

void UGameFeaturesSubsystem::OnAssetManagerCreated();
```

### UEngine::InitializeObjectReferences() 创建并初始化AssetManager
```
void UEngine::InitializeObjectReferences()
{
    // DefaultEngine.ini中配置[/Script/Engine.Engine]
    // AssetManagerClassName=/Script/LyraGame.LyraAssetManager
    AssetManager = NewObject<UAssetManager>(this, SingletonClass);
    AssetManager->StartInitialLoading();
}
```

### UAssetManager::StartInitialLoading() 扫描配置的PrimaryAsset,广播代理
```
void UAssetManager::StartInitialLoading()
{
	ScanPrimaryAssetTypesFromConfig();

	OnAssetManagerCreatedDelegate.Broadcast();
	OnAssetManagerCreatedDelegate.Clear();
}
```

## TODO
```
UGameFeaturesSubsystem::LoadGameFeatureData()
StateProperties.GameFeatureData = Cast<UGameFeatureData>(GameFeatureDataHandle->GetLoadedAsset());
```