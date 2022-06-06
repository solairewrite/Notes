# Lyra_02_创建基础类
## 目录
- [Lyra_02_创建基础类](#lyra_02_创建基础类)
   - [目录](#目录)
   - [模块重命名LyraGame](#模块重命名lyragame)
   - [安装插件](#安装插件)
      - [引擎中勾选插件](#引擎中勾选插件)
         - [GameFeatures](#gamefeatures)
         - [ModularGameplay](#modulargameplay)
         - [GameplayAbilities](#gameplayabilities)
      - [从Lyra项目中复制插件](#从lyra项目中复制插件)
         - [ModularGameplayActors](#modulargameplayactors)
         - [CommonGame](#commongame)
         - [CommonUser](#commonuser)
      - [创建插件](#创建插件)
         - [ShooterCore](#shootercore)
         - [ShooterMaps](#shootermaps)
   - [Gameplay Class的设置](#gameplay-class的设置)
   - [Pawn Class的设置](#pawn-class的设置)
   - [Pawn的创建位置](#pawn的创建位置)
   - [要创建的文件](#要创建的文件)
   - [遇到的问题](#遇到的问题)
      - [B_ShooterGame_Elimination无法引用HeroData_ShooterGame](#b_shootergame_elimination无法引用herodata_shootergame)
      - [WorldSettings无法引用B_ShooterGame_Elimination](#worldsettings无法引用b_shootergame_elimination)

## 模块重命名LyraGame
对项目中各种.Target.cs文件和.build.cs文件的文件名和文件内部的命名进行修改  
模块内部要调用`IMPLEMENT_PRIMARY_GAME_MODULE(FLyraGameMode, LyraGame, "LyraGame");`  

## 安装插件
### 引擎中勾选插件
#### GameFeatures

#### ModularGameplay
以UGameFrameworkComponent为基类,提供如UControllerComponent, UPlayerStateComponent等组件类  
主要提供一些Get函数  

使用GameFrameworkComponentManager,为GameFeature提供增加组件的功能  

#### GameplayAbilities

### 从Lyra项目中复制插件
#### ModularGameplayActors
为GameMode, GameState, PlayerController等提供基类  

PreInitializeComponents(),将Actor注册为组件的接收者  
BeginPlay(),向GFCM发送声明周期事件,广播代理  
EndPlay(),移除添加的组件,广播代理  

对于ModularGameplay模块下的继承自UGameFrameworkComponent的组件,调用对应的生命周期函数  

#### CommonGame
提供PlayerController, LocalPlayer的基类  
为LocalPlayer添加PlayerController, PlayerState, Pawn改变的代理  

#### CommonUser
CommonGame引用此插件  

### 创建插件
#### ShooterCore
在Edit -> Plugins页面,点击Add按钮会弹出创建GameFeature的页面  
选择GameFeature(with C++)  
路径是...\Plugins\GameFeatures,要自己手动创建GameFeatures文件夹  
创建ShooterCore插件,会卡一段时间然后创建成功  

在.uproject文件中Plugins数组添加ShooterCore,否则引擎再次打开时不会显示此插件  
需要在.build.cs文件中添加模块依赖?  

#### ShooterMaps
提供地图,需要在ShooterMaps的GameFeatureData文件中,点击Edit Plugin按钮  
在Dependencies -> Plugins下面添加ShooterCore,才能引用ShooterCore中的文件  

## Gameplay Class的设置
DefaultEngine.ini中设置GlobalDefaultGameMode, WorldSettingsClassName, AssetManagerClassName  
GameState等直接在GameMode构造函数中设置  

## Pawn Class的设置
参考UML  

## Pawn的创建位置
默认在AGameModeBase::PostLogin()中,调用HandleStartingNewPlayer(NewPlayer)来Spawn Pawn  

ALyraGameMode::HandleStartingNewPlayer_Implementation()中判断体验加载完成才能Spawn,实际这里体验没有加载完成,是无法Spawn的  

ALyraGameMode::OnExperienceLoaded()中创建了Pawn  

## 要创建的文件
我这里犯了一个错误



Project Settings -> Asset Manager

ShooterCore的GameFeatureData文件配置AssetManager Primary AssetTypes to Scan  

## 遇到的问题
### B_ShooterGame_Elimination无法引用HeroData_ShooterGame
HeroData_ShooterGame是一个DataAsset,但是我把他创建为蓝图类,导致无法识别  

### WorldSettings无法引用B_ShooterGame_Elimination
```
[AssetLog] E:\Learn\LearnLyra\Content\System\DefaultEditorMap\L_DefaultEditorOverview.umap: 
Illegally references asset /ShooterCore/Experiences/B_ShooterGame_Elimination. 
You may only reference assets from EngineContent, and ProjectContent here. 
(AssetValidator_AssetReferenceRestrictions)
```

应该是游戏本体Content无法引用Plugins中的内用,但是Plugins配置依赖后可以引用其他Plugins的内容  

需要创建ShooterMaps插件,然后在它的GameFeatureData文件中,点击Edit Plugin按钮  
在Dependencies -> Plugins下面添加ShooterCore,才能引用ShooterCore中的文件  
