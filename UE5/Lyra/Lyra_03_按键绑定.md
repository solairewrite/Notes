# Lyra_03_按键绑定
## 目录
- [Lyra_03_按键绑定](#lyra_03_按键绑定)
  - [目录](#目录)
  - [概述](#概述)
    - [核心函数](#核心函数)
    - [整体流程](#整体流程)
  - [创建Class](#创建class)
    - [勾选插件](#勾选插件)
    - [要创建的组件](#要创建的组件)
    - [要创建的蓝图](#要创建的蓝图)
  - [配置位置](#配置位置)
    - [Defaultinput.ini](#defaultinputini)
    - [配置FKey(按键)到InputAction的映射](#配置fkey按键到inputaction的映射)
      - [ShooterCore](#shootercore)
      - [PMI_Default_KBM](#pmi_default_kbm)
      - [IMC_Default_KBM 配置FKey(按键)到InputAction的映射](#imc_default_kbm-配置fkey按键到inputaction的映射)
    - [配置InputAction到GameplayTag的映射](#配置inputaction到gameplaytag的映射)
      - [L_Expanse地图的WorldSettings](#l_expanse地图的worldsettings)
      - [B_ShooterGame_Elimination 游戏体验](#b_shootergame_elimination-游戏体验)
      - [HeroData_ShooterGame 角色配置](#herodata_shootergame-角色配置)
      - [InputData_Hero 配置GameplayTag到InputAction的映射](#inputdata_hero-配置gameplaytag到inputaction的映射)
      - [AbilitySet_ShooterHero](#abilityset_shooterhero)
    - [配置GameplayTag到按键回调函数的映射](#配置gameplaytag到按键回调函数的映射)
      - [对于移动](#对于移动)
      - [对于GameplayAbility](#对于gameplayability)
  - [UGameFeatureAction_AddInputConfig 储存配置的FKey按键到InputAction的映射](#ugamefeatureaction_addinputconfig-储存配置的fkey按键到inputaction的映射)
  - [ULyraHeroComponent::InitializePlayerInput 按键绑定入口](#ulyraherocomponentinitializeplayerinput-按键绑定入口)
    - [ULyraInputComponent::AddInputMappings 绑定FKey到InputAction的映射](#ulyrainputcomponentaddinputmappings-绑定fkey到inputaction的映射)
    - [ULyraInputComponent::BindNativeAction 将UInputAction绑定到代理函数](#ulyrainputcomponentbindnativeaction-将uinputaction绑定到代理函数)

## 概述
### 核心函数
IEnhancedInputSubsystemInterface::AddPlayerMappableConfig, 内置函数,将FKey绑定到UInputAction  
UEnhancedInputComponent::BindAction, 内置函数,将UInputAction绑定到代理函数  

### 整体流程
1. UGameFeatureAction_AddInputConfig::OnGameFeatureActivating  

将IMC_Default_KBM中配置的按键到InputAction的映射,储存到ULyraSettingsLocal::RegisteredInputConfigs  

2. ULyraHeroComponent::InitializePlayerInput  

调用AddInputMappings, BindNativeAction  

3. ULyraInputComponent::AddInputMappings  

获取ULyraSettingsLocal::RegisteredInputConfigs中存储的按键到InputAction的映射  
调用核心函数AddPlayerMappableConfig,将FKey映射到InputAction  

4. ULyraInputComponent::BindNativeAction  

根据GameplayTags,获取InputData_Hero中配置的对应的InputAction,然后直接写死这个Tag对应的回调函数  
调用核心函数BindAction,将UInputAction绑定到代理函数  

## 创建Class
### 勾选插件
Enhanced Input  

### 要创建的组件
ULyraInputComponent: 内置生成,看类设置  
ULyraPawnExtensionComponent: ALyraCharacter构造函数中生成  
ULyraHeroComponent: B_Hero_ShooterMannequin继承的B_Hero_Default蓝图挂载的组件  

### 要创建的蓝图
B_Hero_ShooterMannequin, B_Hero_Default: 角色类  
InputData_Hero: 配置InputTag -> InputAction的映射  
PMI_Default_KBM, IMC_Default_KBM: 配置按键 -> InputAction的映射  

## 配置位置
### Defaultinput.ini
```
DefaultInputComponentClass=/Script/LyraGame.LyraInputComponent
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
```

### 配置FKey(按键)到InputAction的映射
#### ShooterCore
Plugins/GameFeatures/ShooterCore 的 GameFeatureData  
Actions数组中添加GameFeayureAction_AddInputConfig  
里面的InputConfigs数组中包含:  
Config: PMI_Default_KBM  

#### PMI_Default_KBM
引擎内置,继承自UPlayerMappableInputConfig : UPrimaryDataAsset  
Contexts: IMC_Default_KBM  

#### IMC_Default_KBM 配置FKey(按键)到InputAction的映射
继承自引擎内置类 UInputMappingContext : UDataAsset  
其中一个配置是:  
IA_Move到按键WASD的映射  

按键D,Modifiers无配置,代表向X轴正向有输入  
按键A,Modifiers配置Negate,表示向X轴负向输入  
按键W,Modifiers配置Swizzle Input Axis Values,表示重新排列轴的顺序,Order配置YXZ,表示向Y轴正向输入  

UInputModifierNegate: 引擎内置,反转每个轴的输入  
UInputModifierSwizzleAxis: 引擎内置,重排轴向,常用于将X轴转化为Y轴  

### 配置InputAction到GameplayTag的映射
#### L_Expanse地图的WorldSettings
Plugins/GameFeatures/ShooterMaps里面的地图  
DefaultGameplayExperience: B_ShooterGame_Elimination  

#### B_ShooterGame_Elimination 游戏体验
DefaultPawnData: HeroData_ShooterGame  

#### HeroData_ShooterGame 角色配置
PawnClass: B_Hero_ShooterMannequin  
InputConfig: InputData_Hero  

#### InputData_Hero 配置GameplayTag到InputAction的映射
类型: LyraInputConfig  
配置: NativeInputAction和AbilityInputAction的 GameplayTag -> InputAction 映射  

对于NativeInputAction,代码里直接写死Tag对应的Func. 通过这里的配置,找到Tag对应的InputAction绑定Func  
对于AbilityInputAction,代码里Tag是Func的可变参数. 通过这里的配置,按键回调函数的参数为GA对应的InputTag  

UInputAction继承自UDataAsset,由开发者自己在蓝图中创建若干实例,表示一个个的输入事件  
其中一个是:  
InputAction: IA_Move  
InputTag: InputTag.Move  

#### AbilitySet_ShooterHero
类型: LyraAbilitySet  
配置要赋予的GA数组和每个GA对应的InputTag  
在赋予GA时,将InputTag加到AbilitySpec.DynamicAbilityTags  

### 配置GameplayTag到按键回调函数的映射
#### 对于移动
ULyraHeroComponent::InitializePlayerInput()在函数代码中直接写死  
绑定 "InputTag.Move" 到 ULyraHeroComponent::Input_Move()  

#### 对于GameplayAbility
TODO  

## UGameFeatureAction_AddInputConfig 储存配置的FKey按键到InputAction的映射
```
void UGameFeatureAction_AddInputConfig::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    // Pair.Config: PMI_Default_KBM
    // Pair.Config.Contexts: IMC_Default_KBM, 配置按键到InputAction的映射
    for (const FMappableConfigPair& Pair : InputConfigs)
    {
        // 将映射储存到ULyraSettingsLocal::RegisteredInputConfigs
        FMappableConfigPair::ActivatePair(Pair);
    }
}
```

## ULyraHeroComponent::InitializePlayerInput 按键绑定入口
应用FKey到InputAction,InputAction到回调函数的映射  

```
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    UEnhancedInputLocalPlayerSubsystem* Subsystem = LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();

    ULyraInputComponent* LyraIC = CastChecked<ULyraInputComponent>(PlayerInputComponent);

    // PawnData: HeroData_ShooterGame
    // InputConfig: InputData_Hero
    ULyraInputConfig* InputConfig = PawnData->InputConfig;

    // 应用ULyraSettingsLocal::RegisteredInputConfigs中储存的FKey到InputAction的映射
    LyraIC->AddInputMappings(InputConfig, Subsystem);

    // 根据GameplayTag,找到InputData_Hero中配置的InputAction
    // 然后将InputAction绑定到回调函数
    // 这里其实是将GameplayTag到回调函数的映射写死
    LyraIC->BindNativeAction(
        InputConfig, 
        GameplayTags.InputTag_Move, 
        ETriggerEvent::Triggered, 
        this, 
        &ThisClass::Input_Move, 
        /*bLogIfNotFound=*/ false
    );
}
```

### ULyraInputComponent::AddInputMappings 绑定FKey到InputAction的映射
调用核心函数IEnhancedInputSubsystemInterface::AddPlayerMappableConfig,将FKey映射到InputAction  

```
void ULyraInputComponent::AddInputMappings(const ULyraInputConfig* InputConfig, UEnhancedInputLocalPlayerSubsystem* InputSubsystem) const
{
    // 获取ULyraSettingsLocal::RegisteredInputConfigs
    const TArray<FLoadedMappableConfigPair>& Configs = LocalSettings->GetAllRegisteredInputConfigs();

    for (const FLoadedMappableConfigPair& Pair : Configs)
    {
        // 内置函数,应用FKey(按键)到InputAction的映射
        InputSubsystem->AddPlayerMappableConfig(Pair.Config, Options);
    }
}
```

### ULyraInputComponent::BindNativeAction 将UInputAction绑定到代理函数
```
template<class UserClass, typename FuncType>
void ULyraInputComponent::BindNativeAction(
    const ULyraInputConfig* InputConfig, // InputData_Hero
    const FGameplayTag& InputTag, // GameplayTags.InputTag_Move
    ETriggerEvent TriggerEvent, // ETriggerEvent::Triggered,内置枚举先不管
    UserClass* Object, 
    FuncType Func, // 代理函数
    bool bLogIfNotFound
    )
{
    // 根据GameplayTags.InputTag_Move,从InputData_Hero中找到对应移动的InputAction
    if (const UInputAction* IA = InputConfig->FindNativeInputActionForTag(InputTag, bLogIfNotFound))
    {
        // 内置函数,将UInputAction绑定到代理函数
        BindAction(IA, TriggerEvent, Object, Func);
    }
}
```