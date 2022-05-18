# GAS_08_其他
## 自定义GameplayEffectContext
参考中文文档,GASShooter项目  

## 自定义AbilitySystemGlobals
参考GASShooter项目
DefaultGame.ini中设置AbilitySystemGlobalsClassName  

## Batch批处理
参考GASShooter项目Rifle射击,和UML图  

## GameplayCue RPC超出数量限制警告
Attempted to fire NetMulticast_InvokeGameplayCueExecuted_WithParams when no more RPCs are allowed this net update. Max:2  
GAS强制在每次网络更新中最多能有两个相同的GameplayCueRPC,可以通过使用客户端GameplayCue来避免这个问题  
