# UnrealTournament_01
Notes\UML中有UML图  

> 禁用代码优化  

```
// .build.cs
OptimizeCode = CodeOptimization.InShippingBuildsOnly;
```

## 初始化逻辑
### 1.创建GameInstance
```
UGameInstance* UEditorEngine::CreatePIEGameInstance()
{
    // 创建GameInstance
    FStringClassReference GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
	UClass* GameInstanceClass = LoadObject<UClass>(NULL, *GameInstanceClassName.ToString());
    UGameInstance* GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);

    // 初始化GameInstance,创建World
    const FGameInstancePIEResult InitializeResult = GameInstance->InitializeForPlayInEditor(InPIEInstance, GameInstanceParams);

    FWorldContext* const PieWorldContext = GameInstance->GetWorldContext();
    PlayWorld = PieWorldContext->World();
	GWorld = PlayWorld;
	SetPlayInEditorWorld( PlayWorld );

    // 创建ViewportClient
    UGameViewportClient* ViewportClient = NULL;
	ULocalPlayer *NewLocalPlayer = NULL;

    ViewportClient = NewObject<UGameViewportClient>(this, GameViewportClientClass);
    ViewportClient->Init(*PieWorldContext, GameInstance, bCreateNewAudioDevice);

    GameViewport = ViewportClient;
    GameViewport->bIsPlayInEditorViewport = true;
    PieWorldContext->GameViewport = ViewportClient;

    // 创建LocalPlayer,内部调用UGameInstance::CreateLocalPlayer,这里没有Spawn PlayerController
    NewLocalPlayer = ViewportClient->SetupInitialLocalPlayer(Error);

    // 开始GameInstance,创建GameMode,PlayerController
    const FGameInstancePIEResult StartResult = GameInstance->StartPlayInEditorGameInstance(NewLocalPlayer, GameInstanceParams);
}
```

### 2.创建WorldContext,World
+ 调用 UGameInstance::Init  
```
FGameInstancePIEResult UGameInstance::InitializeForPlayInEditor()
{
    WorldContext = &EditorEngine->CreateNewWorldContext(EWorldType::PIE);
    WorldContext->OwningGameInstance = this;

    // 标准的PIE路径,直接复制EditorWorld
    NewWorld = EditorEngine->CreatePIEWorldByDuplication(*WorldContext, EditorEngine->EditorWorld, PIEMapName);

    NewWorld->SetGameInstance(this);
	WorldContext->SetCurrentWorld(NewWorld);

    // 可重写,给自定义GameInstance设置的机会
    Init();
}
```

```
UWorld* UEditorEngine::CreatePIEWorldByDuplication()
{
    UWorld* NewPIEWorld = NULL;
    NewPIEWorld = CastChecked<UWorld>( StaticDuplicateObject(
			InWorld,				// Source root
			PlayWorldPackage,		// Destination root
			...
			) );
    return NewPIEWorld;
}
```

### 3.创建GameViewportClient,LocalPlayer  
```
ULocalPlayer* UGameInstance::CreateLocalPlayer(bool bSpawnActor)
{
    ULocalPlayer* NewPlayer = NULL;
    NewPlayer = NewObject<ULocalPlayer>(GetEngine(), GetEngine()->LocalPlayerClass);

    if(bSpawnActor)
    {
        // 创建PlayerController
        NewPlayer->SpawnPlayActor("", OutError, GetWorld());
    }
}
```

### 4.创建GameMode,GameState,PlayerController,PlayerState
```
UGameInstance::StartPlayInEditorGameInstance()
{
    UWorld* const PlayWorld = GetWorld();

    FURL URL;

    // 设置了FString FURL::Map为当前地图
    URL = FURL(NULL, *EditorEngine->BuildPlayWorldURL(*PIEMapName, Params.bStartInSpectatorMode), TRAVEL_Absolute);

    // 创建GameMode
    PlayWorld->SetGameMode(URL)

    // 调用AGameModeBase::InitGame
    // 创建GameState
    PlayWorld->InitializeActorsForPlay(URL);

    // Spawn PlayerController
    LocalPlayer->SpawnPlayActor(URL.ToString(1), Error, PlayWorld)

    // 游戏开始前,阻塞stream level
    GEngine->BlockTillLevelStreamingCompleted(PlayWorld);

    // TOOD: 注意这里的BeginPlay顺序
    PlayWorld->BeginPlay();
}
```

> 创建GameMode  
```
bool UWorld::SetGameMode()
{
    AuthorityGameMode = GetGameInstance()->CreateGameModeForURL(InURL);
}
```

```
AGameModeBase* UGameInstance::CreateGameModeForURL(FURL InURL)
{
    TSubclassOf<AGameModeBase> GameClass = Settings->DefaultGameMode;

    // 有根据URL中的地图名设置GameClass的逻辑,因为UT没走这里,所以先略过

    if (!GameClass)
    {
        // 这是ProjectSettings中设置的DefaultGameMode
        GameClass = LoadClass<AGameModeBase>(nullptr, *UGameMapsSettings::GetGlobalDefaultGameMode());
    }

    return World->SpawnActor<AGameModeBase>(GameClass, SpawnInfo);
}
```

> 创建GameState  
+ 调用 AGameModeBase::InitGame  
```
void UWorld::InitializeActorsForPlay()
{
    if( !AreActorsInitialized() )
    {
        if (AuthorityGameMode && !AuthorityGameMode->IsActorInitialized())
		{
            // AUTBaseGameMode::InitGame中,设置了DefaultPawnClass
			AuthorityGameMode->InitGame( FPaths::GetBaseFilename(InURL.Map), Options, Error );
		}

        // Route various initialization functions and set volumes.
		for( int32 LevelIndex=0; LevelIndex<Levels.Num(); LevelIndex++ )
		{
			ULevel*	const Level = Levels[LevelIndex];
            // 创建GameState
			Level->RouteActorInitialize();
		}
    }
}
```

```
void ULevel::RouteActorInitialize()
{
	// Send PreInitializeComponents and collect volumes.
	for( int32 Index = 0; Index < Actors.Num(); ++Index )
	{
		AActor* const Actor = Actors[Index];
		if( Actor && !Actor->IsActorInitialized() )
		{
			Actor->PreInitializeComponents();
		}
	}
}
```

```
void AGameModeBase::PreInitializeComponents()
{
	Super::PreInitializeComponents();

    GameState = GetWorld()->SpawnActor<AGameStateBase>(GameStateClass, SpawnInfo);
}
```