# Lyra_01_大纲
## 目录
- [Lyra_01_大纲](#lyra_01_大纲)
    - [目录](#目录)
    - [章节列表](#章节列表)
    - [配置](#配置)
    - [TODO](#todo)
    - [开枪](#开枪)

## 章节列表
[Lyra_02_创建基础类](Lyra_02_创建基础类.md)  
[Lyra_03_按键绑定](Lyra_03_按键绑定.md)  

## 配置
默认地图: L_DefaultEditorOverview  
游戏地图: L_Expanse  
设置Bot数量: B_ShooterBotSpawner, ULyraBotCreationComponent::ServerCreateBots  

WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
GlobalDefaultGameMode=/Game/B_LyraGameMode.B_LyraGameMode_C

## TODO
LyraGameFeaturePolicy  
注意下AssetManager的配置

创建 B_LyraGameMode : LyraGameMode  
创建 B_Hero_ShooterMannequin (ShooterCore插件) : B_Hero_Default : Character_Default : LyraCharacter  

## 开枪
GA_Weapon_Fire_Rifle_Auto  
GE_Damage_RifleAuto  
GA_Weapon_Fire_Pistol  
InputTag.Weapon.Fire  