# Lyra_03_按键绑定
## 目录
- [Lyra_03_按键绑定](#lyra_03_按键绑定)
    - [目录](#目录)
    - [按键绑定](#按键绑定)
    - [勾选插件](#勾选插件)
    - [要创建的组件](#要创建的组件)
    - [要创建的蓝图](#要创建的蓝图)

## 按键绑定
GA_Weapon_Fire_Rifle_Auto
GE_Damage_RifleAuto
GA_Weapon_Fire_Pistol
InputTag.Weapon.Fire

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
