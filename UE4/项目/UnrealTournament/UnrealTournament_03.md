# UnrealTournament_03
## 设置主要Class

+ DefaultEngine.ini  
```
UCLASS(config=Engine, defaultconfig)
class UGameMapsSettings
{
	UPROPERTY(config)
	FStringClassReference GameInstanceClass;

	UPROPERTY(config)
	FStringClassReference GlobalDefaultGameMode;
}

UCLASS(config=Engine, defaultconfig)
class UEngine
{
	UPROPERTY(globalconfig)
	FStringClassReference GameViewportClientClassName;

	UPROPERTY(globalconfig)
	FStringClassReference LocalPlayerClassName;
}
```

> 设置WorldSetting Class  
```
[/Script/Engine.Engine]
WorldSettingsClassName=/Script/UnrealTournament.UTWorldSettings
```

> 设置GameInstance Class  
```
[/Script/EngineSettings.GameMapsSettings]
GameInstanceClass=/Script/UnrealTournament.UTGameInstance

// UEditorEngine::CreatePIEGameInstance
FStringClassReference GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
UClass* GameInstanceClass = LoadObject<UClass>(NULL, *GameInstanceClassName.ToString())
```

> 设置GameViewportClient Class  
```
[/Script/Engine.Engine]
GameViewportClientClassName=/Script/UnrealTournament.UTGameViewportClient

// UEngine::InitializeObjectReferences()
LoadEngineClass<UGameViewportClient>(GameViewportClientClassName, GameViewportClientClass);
```

> 设置LocalPlayer Class  
```
LocalPlayerClassName=/Script/UnrealTournament.UTLocalPlayer

// UEngine::InitializeObjectReferences()
LoadEngineClass<ULocalPlayer>(LocalPlayerClassName, LocalPlayerClass);
```

> 设置GameMode Class  

如果WorldSettings中未设置GameModeOverride  
在ProjectSettings中设置DefaultGameMode  
```
// DefaultEngine.ini
[/Script/EngineSettings.GameMapsSettings]
GlobalDefaultGameMode=/Script/MyUnrealTournament.UTDMGameMode

// UGameInstance::CreateGameModeForURL
GameClass = LoadClass<AGameModeBase>(nullptr, *UGameMapsSettings::GetGlobalDefaultGameMode());
```

> 设置Pawn Class  
```
AUTGameMode::AUTGameMode()
{
    PlayerPawnObject = FStringAssetReference(TEXT("/Game/RestrictedAssets/Blueprints/DefaultCharacter.DefaultCharacter_C"));
}

void AUTBaseGameMode::InitGame( const FString& MapName, const FString& Options, FString& ErrorMessage )
{
	if (!PlayerPawnObject.IsNull())
	{
		DefaultPawnClass = Cast<UClass>(StaticLoadObject(UClass::StaticClass(), NULL, *PlayerPawnObject.ToStringReference().ToString(), NULL, LOAD_NoWarn));
	}

    Super::InitGame(MapName, Options, ErrorMessage);
}
```