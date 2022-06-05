# Lyra_01_大纲
## 目录
- [Lyra_01_大纲](#lyra_01_大纲)
    - [目录](#目录)
    - [章节列表](#章节列表)
    - [TODO](#todo)
    - [配置](#配置)
    - [按键绑定](#按键绑定)

## 章节列表
[Lyra_02_创建基础类](Lyra_02_创建基础类.md)  

## TODO
LyraGameFeaturePolicy  
注意下AssetManager的配置

创建 ShooterCore插件  
创建 B_ShooterGame_Elimination  
创建 HeroData_ShooterGame  
创建 B_LyraGameMode : LyraGameMode  
创建 B_Hero_ShooterMannequin (ShooterCore插件) : B_Hero_Default : Character_Default : LyraCharacter  

配置  
```
TArray<FString> ULyraExperienceDefinition::GameFeaturesToEnable;
TObjectPtr<const ULyraPawnData> DefaultPawnData;
ALyraWorldSettings::DefaultGameplayExperience
```

## 配置
地图L_Expanse  
设置Bot数量: B_ShooterBotSpawner, ULyraBotCreationComponent::ServerCreateBots  

WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
GlobalDefaultGameMode=/Game/B_LyraGameMode.B_LyraGameMode_C

## 按键绑定
GA_Weapon_Fire_Rifle_Auto
GE_Damage_RifleAuto
GA_Weapon_Fire_Pistol
InputTag.Weapon.Fire

ULyraInputComponent::BindNativeAction
InputData_Hero
HeroData_ShooterGame
ULyraInputComponent::BindAbilityActions
ULyraInputComponent::AddInputMappings