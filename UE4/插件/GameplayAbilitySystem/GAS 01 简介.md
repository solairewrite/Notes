# GAS 01 简介
[中文文档](https://github.com/BillEliot/GASDocumentation_Chinese)  
[我自己的GASDoc项目](https://git.code.tencent.com/solairewrite/GASDoc)  
[自己的文档](https://git.code.tencent.com/solairewrite/Notes/tree/master/UE4/%E6%8F%92%E4%BB%B6/GameplayAbilitySystem)  
项目中含有UML图 

## 调试
showdebug abilitysystem 显示3页属性  
AbilitySystem.Debug.NextCategory 翻页  
PageUp / PageDown 更换调试对象

## 启用GAS插件
在UE4编辑器中,Plugins,启用Gameplay Abilities  
```
// GASDoc.build.cs
PrivateDependencyModuleNames.AddRange(new string[] {
            "GameplayAbilities",
            "GameplayTags",
            "GameplayTasks",
        });
```

## GAS概念
ASC附加的Actor为`OwnerActor`,ASC的物理代表Actor为`AvatarActor`  
OwnerActor和AvatarActor可以是同一个Actor,也可以是不同的Actor  
如果玩家会重生,OwnerActor为PlayerState,AvatarActor为Character  

同步模式  
| 同步模式 | 何时使用 | 同步规则 |
| - | - | - |
| Full | 单机 | |
| Mixed | 联机,玩家 | GE只同步到所属客户端,Tag和GC同步到所有客户端 |
| Minimal | 联机,AI | GE不同步,Tag和GC同步到所有客户端 |

Mixed模式需要设置OwnerActor的Owner是Controller,PlayerState的Owner模式是Controller  
对于Character,需要在PossessedBy()设置Owner为Controller  

