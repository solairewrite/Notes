# Lyra_03_按键绑定
## 目录
- [Lyra_03_按键绑定](#lyra_03_按键绑定)
    - [目录](#目录)
    - [按键绑定](#按键绑定)
    - [勾选插件](#勾选插件)
    - [要创建的组件](#要创建的组件)
    - [要创建的蓝图](#要创建的蓝图)
    - [配置位置](#配置位置)
        - [配置InputAction到FKey(按键)的映射](#配置inputaction到fkey按键的映射)
            - [ShooterCore](#shootercore)
            - [PMI_Default_KBM](#pmi_default_kbm)
            - [IMC_Default_KBM 配置InputAction到FKey(按键)的映射](#imc_default_kbm-配置inputaction到fkey按键的映射)
        - [配置InputAction到GameplayTag的映射](#配置inputaction到gameplaytag的映射)
            - [L_Expanse地图的WorldSettings](#l_expanse地图的worldsettings)
            - [B_ShooterGame_Elimination 游戏体验](#b_shootergame_elimination-游戏体验)
            - [HeroData_ShooterGame 角色配置](#herodata_shootergame-角色配置)
            - [InputData_Hero 配置InputAction到GameplayTag的映射](#inputdata_hero-配置inputaction到gameplaytag的映射)
        - [配置GameplayTag到按键回调函数的映射](#配置gameplaytag到按键回调函数的映射)
            - [对于移动](#对于移动)
            - [对于GameplayAbility](#对于gameplayability)
    - [核心函数](#核心函数)
    - [TODO](#todo)
    - [RebuildControlMappings 应该是核心函数,要看的](#rebuildcontrolmappings-应该是核心函数要看的)
    - [ULyraHeroComponent::InitializePlayerInput](#ulyraherocomponentinitializeplayerinput)
        - [ULyraInputComponent::AddInputMappings 绑定InputAction到FKey(按键)的映射](#ulyrainputcomponentaddinputmappings-绑定inputaction到fkey按键的映射)
        - [ULyraInputComponent::BindNativeAction](#ulyrainputcomponentbindnativeaction)
    - [UGameFeatureAction_AddInputConfig 绑定InputAction到FKey按键](#ugamefeatureaction_addinputconfig-绑定inputaction到fkey按键)

## 按键绑定

## 勾选插件
Enhanced Input  

## 要创建的组件
ULyraInputComponent: 内置生成,看类设置  
ULyraPawnExtensionComponent: ALyraCharacter构造函数中生成  
ULyraHeroComponent: B_Hero_ShooterMannequin继承的B_Hero_Default蓝图挂载的组件  

## 要创建的蓝图
B_Hero_ShooterMannequin, B_Hero_Default: 角色类  
InputData_Hero: 配置InputAction -> InputTag的映射  
PMI_Default_KBM, IMC_Default_KBM: 配置按键 -> 字符串的映射?  

## 配置位置
ULyraHeroComponent::DefaultInputConfigs: 储存UInputAction到FKey的映射  
InputData_Hero: 储存FGameplayTag到UInputAction的映射  

### 配置InputAction到FKey(按键)的映射
#### ShooterCore
Plugins/GameFeatures/ShooterCore 的 GameFeatureData  
Actions数组中添加GameFeayureAction_AddInputConfig  
里面的InputConfigs数组中包含的其中一个:  
Config: PMI_Default_KBM  

#### PMI_Default_KBM
引擎内置,继承自UPlayerMappableInputConfig : UPrimaryDataAsset  
Contexts: IMC_Default_KBM  

#### IMC_Default_KBM 配置InputAction到FKey(按键)的映射
引擎内置,继承自UInputMappingContext : UDataAsset  
其中一个配置是:  
IA_Move到按键WASD的映射  

<font color=red>问题: WASD是怎么确认前后左右的?</font>  
<font color=red>问题: PlayerMappableOptions里面的MoveForward有用到吗?</font>  

难道按键D,啥也没配置,就代表向X轴正向有输入  
按键A,Modifiers配置Negate,表示向X轴负向输入  
按键W,Modifiers配置Swizzle Input Axis Values,表示重新排列轴的顺序,Order配置YXZ,表示向Y轴正向输入  
UInputModifierNegate: 引擎内置,反转每个轴的输入  
UInputModifierSwizzleAxis: 重排轴向,常用于将X轴转化为Y轴  

### 配置InputAction到GameplayTag的映射
#### L_Expanse地图的WorldSettings
Plugins/GameFeatures/ShooterMaps里面的地图  
DefaultGameplayExperience: B_ShooterGame_Elimination  

#### B_ShooterGame_Elimination 游戏体验
DefaultPawnData: HeroData_ShooterGame  

#### HeroData_ShooterGame 角色配置
PawnClass: B_Hero_ShooterMannequin  
InputConfig: InputData_Hero  

#### InputData_Hero 配置InputAction到GameplayTag的映射
UInputAction继承自UDataAsset,它是在蓝图中创建的一个DataAsset实例,配置输入事件的属性  
其中一个是:  
InputAction: IA_Move  
InputTag: InputTag.Move  

### 配置GameplayTag到按键回调函数的映射
#### 对于移动
ULyraHeroComponent::InitializePlayerInput()在函数代码中直接写死  
绑定 "InputTag.Move" 到 ULyraHeroComponent::Input_Move()  

#### 对于GameplayAbility
TODO  

## 核心函数
IEnhancedInputSubsystemInterface::AddPlayerMappableConfig, 内置函数,将UInputAction绑定到FKey  
UEnhancedInputComponent::BindAction, 内置函数,将UInputAction绑定到代理函数

## TODO
InputAction具体是什么  
UEnhancedInputLocalPlayerSubsystem* Subsystem 具体是什么,是接口?还是继承了?怎么创建的  
FLyraGameplayTags  怎么创建的  
ULyraSettingsLocal 创建  
ULyraHeroComponent::DefaultInputConfigs 设置了什么,啥也没配置?  

UEngine::InitializeObjectReferences和StartPlayInEditorGameInstance的先后顺序  

## RebuildControlMappings 应该是核心函数,要看的
参考UT_04_按键绑定

## ULyraHeroComponent::InitializePlayerInput
```
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    UEnhancedInputLocalPlayerSubsystem* Subsystem = LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();

    // 将ULyraHeroComponent::DefaultInputConfigs中配置的按键存入ULyraSettingsLocal::RegisteredInputConfigs
    for (const FMappableConfigPair& Pair : DefaultInputConfigs)
    {
        FMappableConfigPair::ActivatePair(Pair);
    }

    ULyraInputComponent* LyraIC = CastChecked<ULyraInputComponent>(PlayerInputComponent);

    // PawnData: HeroData_ShooterGame
    // InputConfig: InputData_Hero
    ULyraInputConfig* InputConfig = PawnData->InputConfig;

    // ???
    LyraIC->AddInputMappings(InputConfig, Subsystem);

    // ???
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

### ULyraInputComponent::AddInputMappings 绑定InputAction到FKey(按键)的映射
```
void ULyraInputComponent::AddInputMappings(const ULyraInputConfig* InputConfig, UEnhancedInputLocalPlayerSubsystem* InputSubsystem) const
{
    // 获取ULyraSettingsLocal::RegisteredInputConfigs
    const TArray<FLoadedMappableConfigPair>& Configs = LocalSettings->GetAllRegisteredInputConfigs();

    for (const FLoadedMappableConfigPair& Pair : Configs)
    {
        // 内置函数,应用映射
        // ULyraHeroComponent::DefaultInputConfigs中配置的按键和事件的映射???
        InputSubsystem->AddPlayerMappableConfig(Pair.Config, Options);
    }
}
```

### ULyraInputComponent::BindNativeAction
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

## UGameFeatureAction_AddInputConfig 绑定InputAction到FKey按键
```
void UGameFeatureAction_AddInputConfig::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    for (const FMappableConfigPair& Pair : InputConfigs)
    {
        FMappableConfigPair::ActivatePair(Pair);
    }
}
```