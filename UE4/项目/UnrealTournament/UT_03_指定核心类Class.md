# UT_03_指定核心类Class
## 目录
- [UT_03_指定核心类Class](#ut_03_指定核心类class)
	- [目录](#目录)
	- [DefaultEngine.ini中配置的Class](#defaultengineini中配置的class)
		- [LocalPlayer](#localplayer)
		- [GameMode](#gamemode)
	- [代码中指定的Class](#代码中指定的class)
		- [Pawn](#pawn)

## DefaultEngine.ini中配置的Class
### LocalPlayer
```
FSoftClassPath UEngine::LocalPlayerClassName;

LocalPlayerClassName=/Script/MyUnrealTournament.UTLocalPlayer
```

### GameMode
没在WorldSettings中设置,而是在ProjectSettings中设置  

```
FSoftClassPath UGameMapsSettings::GlobalDefaultGameMode;

GlobalDefaultGameMode=/Script/MyUnrealTournament.UTDMGameMode
```

## 代码中指定的Class
### Pawn
```
AUTGameMode::AUTGameMode()
{
   PlayerPawnObject = FSoftObjectPath(TEXT("/Game/RestrictedAssets/Blueprints/DefaultCharacter.DefaultCharacter_C"));
}

void AUTBaseGameMode::InitGame()
{
	DefaultPawnClass = Cast<UClass>(StaticLoadObject(UClass::StaticClass(), NULL, *PlayerPawnObject.ToString(), NULL, LOAD_NoWarn));
}
```