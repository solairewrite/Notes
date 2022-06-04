# Lyra_01_大纲
## 目录
- [Lyra_01_大纲](#lyra_01_大纲)
    - [目录](#目录)
    - [章节列表](#章节列表)
    - [TODO](#todo)

## 章节列表

## TODO
LyraGameFeaturePolicy  

设置CurrentExperience
B_ShooterGame_Elimination
herodata_shootergame

ULyraExperienceManagerComponent::ServerSetCurrentExperience
ALyraGameMode::OnMatchAssignmentGiven
ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne 定时器?122行有效

ALyraPlayerState::SetPawnData
ALyraPlayerState::OnExperienceLoaded
ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted 341
	OnExperienceLoaded 绑定 
ULyraExperienceManagerComponent::OnGameFeaturePluginLoadComplete
==ULyraExperienceManagerComponent::OnExperienceLoadComplete
UGameFeaturesSubsystem::ChangeGameFeatureTargetStateComplete 
	CompleteDelegate代理怎么绑定的
或
ALyraPlayerState::OnExperienceLoaded
ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded
ALyraPlayerState::PostInitializeComponents

创建类: 
B_LyraGameMode : LyraGameMode
B_Hero_ShooterMannequin (ShooterCore插件) : B_Hero_Default : Character_Default : LyraCharacter

ULyraPawnData
注意下AssetManager的配置

插件 ShooterCore
ShooterMaps / Maps / L_Expanse