# UnrealTournament_04
# 按键绑定
## 关键类
### UTLocalPlayer
在构造时,创建UTProfileSettings  

### UTProfileSettings
`ResetProfile,GetDefaultGameActions`定义按键如何绑定,存储在`TArray<FKeyConfigurationInfo> GameActions;`  
`ApplyInputSettings`移除按键绑定,然后添加GameActions中的按键,最后调用ForceRebuildingKeyMaps应用按键  

### UPlayerInput
内置类,`TArray<struct FInputAxisKeyMapping> AxisMappings;`存储玩家自定义的键位  
`AddAxisMapping`添加键位  
`ForceRebuildingKeyMaps`根据缓存的AxisMappings重新绑定按键  
`ConditionalBuildKeyMappings`由PlayerController每帧调用,实际进行绑定操作  

### AUTBasePlayerController
`InitInputSystem`创建自定义输入`NewObject<UUTPlayerInput>`  
`SetupInputComponent`进行按键绑定  

## 代码分析
```
// Object.h
// 在C++构造函数之后,并且属性已初始化后调用
virtual void PostInitProperties();
```

创建LocalPlayer时  
`NewObject<ULocalPlayer>`内部会调用`PostInitProperties()`  

```
void UUTLocalPlayer::PostInitProperties()
{
	Super::PostInitProperties();

    LoadLocalProfileSettings();
}
```

创建PlayerController时  
```
APlayerController* UWorld::SpawnPlayActor(UPlayer* NewPlayer)
{
    APlayerController* const NewPlayerController = GameMode->Login(NewPlayer);
    // 初始化按键输入
    NewPlayerController->SetPlayer(NewPlayer);
    GameMode->PostLogin(NewPlayerController);
}
```

```
void APlayerController::SetPlayer( UPlayer* InPlayer )
{
    Player = InPlayer;
	InPlayer->PlayerController = this;
    InitInputSystem();
}
```

```
void APlayerController::InitInputSystem()
{
    // 创建PlayerInput
    PlayerInput = NewObject<UUTPlayerInput>();

    SetupInputComponent();
}

void AUTPlayerController::SetupInputComponent()
{
    UUTProfileSettings* ProfileSettings = GetProfileSettings();
    // 应用配置的按键绑定
    ProfileSettings->ApplyInputSettings(Cast<UUTLocalPlayer>(Player));

    // 创建InputComponent
    InputComponent = NewObject<UInputComponent>();

    // 具体的按键绑定
    InputComponent->BindAxis("MoveForward", this, &AUTPlayerController::MoveForward);
}
```

### 原理
`TArray<FInputAxisBinding> UInputComponent::AxisBindings`中存储了所有的按键  
`SetupInputComponent`中调用`BindAxis`向`AxisBindings`中加入按键绑定的代理函数  
PlayerController在Tick函数中,每帧遍历`AxisBindings`中绑定的代理函数,进行调用  

### 绑定具体的按键
```
FInputAxisBinding& BindAxis( const FName AxisName, UserClass* Object, FMethodPtr Func )
{
    FInputAxisBinding AB( AxisName );
	AB.AxisDelegate.BindDelegate(Object, Func);
	AxisBindings.Emplace(MoveTemp(AB));
}
```

### 每帧调用按键绑定代理
1. Tick中调用UPlayerInput::ProcessInputStack  
```
void APlayerController::TickActor( float DeltaSeconds)
{
    PlayerTick(DeltaSeconds);
}

void APlayerController::PlayerTick( float DeltaTime )
{
    TickPlayerInput(DeltaTime, DeltaTime == 0.f);
}

void APlayerController::TickPlayerInput(const float DeltaSeconds, const bool bGamePaused)
{
    ProcessPlayerInput(DeltaSeconds, bGamePaused);
}

void APlayerController::ProcessPlayerInput(const float DeltaTime, const bool bGamePaused)
{
    PlayerInput->ProcessInputStack(InputStack, DeltaTime, bGamePaused);
}
```

2. 调用按键绑定代理函数  
```
void UPlayerInput::ProcessInputStack(const TArray<UInputComponent*>& InputComponentStack)
{
    // 绑定按键
    ConditionalBuildKeyMappings();

    static TArray<FAxisDelegateDetails> AxisDelegates;

    // 断点看有3个输入组件: PawnInputComponent,PC_InputComponent,GameplayDebug_Input
    // PC_InputComponent是实际生效的组件

    int32 StackIndex = InputComponentStack.Num()-1;
    for ( ; StackIndex >= 0; --StackIndex)
    {
        UInputComponent* const IC = InputComponentStack[StackIndex];
        if(IC)
        {
            // 遍历轴向绑定,累积轴向值
            for (FInputAxisBinding& AB : IC->AxisBindings)
			{
				AB.AxisValue = DetermineAxisValue(AB, bGamePaused, KeysToConsume);
				if (AB.AxisDelegate.IsBound())
				{
					AxisDelegates.Emplace(FAxisDelegateDetails(AB.AxisDelegate, AB.AxisValue));
				}
			}
        }
    }

    // 调用按键代理函数
    for (const FAxisDelegateDetails& Details : AxisDelegates)
	{
		if (Details.Delegate.IsBound())
		{
			Details.Delegate.Execute(Details.Value);
		}
	}
}
```