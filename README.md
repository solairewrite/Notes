# UE4开发笔记
阳光明媚的日子敲代码很开心,很放松

## 要添加的知识点
渲染管线shader  
帧同步  
性能优化  
AI源码  
slua的原理,UObject怎么传入lua  
UObject实现了哪些功能  
寻路,光照  

## TODO
### 1.AI模块  
1. 系统看`UBehaviorTreeComponent::OnTaskFinished`

2. `DoDecoratorsAllowExecution`, `FindCommonParent`

### 2.GAS插件  
GAS的文章主要分析源码,绘制uml  
不在意大家都已经分析过的基础文档  
字典直接可查,重在理解原理  

0. UAbilityTask_WaitTargetData
wait input release

0. 输入绑定  

1. 应用GE UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(),GE对属性的修改  

2. 服务器激活GA成功时,通知客户端更新ActivationInfo
```
UAbilitySystemComponent::InternalTryActivateAbility()
{
    ClientActivateAbilitySucceed(Handle, ActivationInfo.GetActivationPredictionKey());
}
```

3. 服务器激活GA失败时,终止客户端的GA,并撤销所有预测的修改
```
UAbilitySystemComponent::InternalServerTryActivateAbility()
{
    ClientActivateAbilityFailed(Handle, PredictionKey.Current);
}
```

4. 被动技能重写UGameplayAbility::OnAvatarSet

5. 内置GA

6. FActiveGameplayEffect::PostReplicatedAdd 堆栈



## 疑问

2. Task的Service先于Task执行吗,是的

3. UBehaviorTreeComponent::UpdateInstanceId 里面的 InstanceId.Path 是逆序吗
    这个path可能只是bool operator==(const FBehaviorTreeInstanceId& Other)用来判断两颗子树是不是同一颗

