# Lyra_02_创建基础类
## 目录
- [Lyra_02_创建基础类](#lyra_02_创建基础类)
   - [目录](#目录)
   - [模块重命名LyraGame](#模块重命名lyragame)
   - [引擎中勾选插件](#引擎中勾选插件)
      - [GameFeatures](#gamefeatures)
      - [ModularGameplay](#modulargameplay)
      - [GameplayAbilities](#gameplayabilities)
   - [从Lyra项目中复制插件](#从lyra项目中复制插件)
      - [ModularGameplayActors](#modulargameplayactors)
      - [CommonGame](#commongame)
      - [CommonUser](#commonuser)
   - [Gameplay Class的设置](#gameplay-class的设置)
   - [Pawn Class的设置](#pawn-class的设置)

## 模块重命名LyraGame
对项目中各种.Target.cs文件和.build.cs文件的文件名和文件内部的命名进行修改  
模块内部要调用`IMPLEMENT_PRIMARY_GAME_MODULE(FLyraGameMode, LyraGame, "LyraGame");`  

## 引擎中勾选插件
### GameFeatures

### ModularGameplay
以UGameFrameworkComponent为基类,提供如UControllerComponent, UPlayerStateComponent等组件类  
主要提供一些Get函数  

使用GameFrameworkComponentManager,为GameFeature提供增加组件的功能  

### GameplayAbilities  

## 从Lyra项目中复制插件
### ModularGameplayActors
为GameMode, GameState, PlayerController等提供基类  

PreInitializeComponents(),将Actor注册为组件的接收者  
BeginPlay(),向GFCM发送声明周期事件,广播代理  
EndPlay(),移除添加的组件,广播代理  

对于ModularGameplay模块下的继承自UGameFrameworkComponent的组件,调用对应的生命周期函数  

### CommonGame
提供PlayerController, LocalPlayer的基类  
为LocalPlayer添加PlayerController, PlayerState, Pawn改变的代理  

### CommonUser
CommonGame引用此插件  

## Gameplay Class的设置
DefaultEngine.ini中设置GlobalDefaultGameMode, WorldSettingsClassName, AssetManagerClassName  
GameState等直接在GameMode构造函数中设置  

## Pawn Class的设置
