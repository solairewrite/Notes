# Lyra_01_大纲
## 目录
- [Lyra_01_大纲](#lyra_01_大纲)
  - [目录](#目录)
  - [章节列表](#章节列表)
  - [配置](#配置)
  - [开枪](#开枪)
  - [TODO](#todo)

## 章节列表
[Lyra_02_创建基础类](Lyra_02_创建基础类.md)  
[Lyra_03_按键绑定](Lyra_03_按键绑定.md)  
[Lyra_04_跳跃GA](Lyra_04_跳跃GA.md)  

## 配置
默认地图: L_DefaultEditorOverview  
游戏地图: L_Expanse  
设置Bot数量: B_ShooterBotSpawner, ULyraBotCreationComponent::ServerCreateBots  

WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
GlobalDefaultGameMode=/Game/B_LyraGameMode.B_LyraGameMode_C

## 开枪
GA_Weapon_Fire_Rifle_Auto  
GE_Damage_RifleAuto  
GA_Weapon_Fire_Pistol  
InputTag.Weapon.Fire  

## TODO
LyraGameFeaturePolicy  

创建 B_LyraGameMode : LyraGameMode  
创建 B_Hero_ShooterMannequin (ShooterCore插件) : B_Hero_Default : Character_Default : LyraCharacter  
  
按下W时ULyraHeroComponent::Input_Move的堆栈  
RebuildControlMappings核心函数  

IMC_Default_KBM 里面有几个 IA_Look_Mouse 的修改  