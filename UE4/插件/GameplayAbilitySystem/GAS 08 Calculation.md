# GAS 08 Calculation
参考`UGDDamageExecCalc::Execute_Implementation`  

## 触发
应用GE时,`Executions`数组中的`CalculationClass`会触发  

## 修改属性
```
UGDDamageExecCalc::Execute_Implementation(
    ..., OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput)
{
    // 这会触发AttributeSet中,Damage的修改,在PostGameplayEffectExecute中进一步修改Health
    OutExecutionOutput.AddOutputModifier(...);
}
```