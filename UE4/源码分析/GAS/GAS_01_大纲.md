# GAS_01_大纲
## 目录
- [GAS_01_大纲](#gas_01_大纲)
    - [目录](#目录)
    - [章节列表](#章节列表)
    - [UML](#uml)
    - [调试](#调试)
    - [TODO](#todo)

## 章节列表
[GAS中文文档](https://github.com/BillEliot/GASDocumentation_Chinese)  

[GAS_02_AbilityTask](GAS_02_AbilityTask.md)  
[GAS_03_GA](GAS_03_GA.md)  
[GAS_04_GE_01_应用GE](GAS_04_GE_01_应用GE.md)  
[GAS_04_GE_02_修改属性](GAS_04_GE_02_修改属性.md)  
[GAS_04_GE_03_移除GE](GAS_04_GE_03_移除GE.md)  
[GAS_05_TargetData](GAS_05_TargetData.md)  
[GAS_06_Prediction](GAS_06_Prediction.md)  
[GAS_07_GameplayTag](GAS_07_GameplayTag.md)  

## UML
UE4 GAS 01 类图  
UE4 GAS 02 流程图  

## 调试
`showdebug abilitysystem`

`AbilitySystem.Debug.NextCategory`

## TODO
TargetData结构,可以传Tag到服务器吗  
即等待客户端事件  
向服务器传Location,可以直接用TargetData那一套吗    

AGameplayAbilityTargetActor::ConfirmTargeting() 里面用到了通用事件  
AGameplayAbilityTargetActor::BindToConfirmCancelInputs() 查看它是怎么绑定代理的  

加一篇代理

wait net sync, SendGameplayEvent  

回顾Prediction  
