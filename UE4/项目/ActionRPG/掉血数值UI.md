# 掉血数值UI
掉血数值向上飘的效果  

+ 创建UMG  
WB_DamageNumber: UserWidget  

+ 创建组件  
WC_DamageText: WidgetComponent  
&emsp;设置 WidgetClass: WB_DamageNumber  

+ 使用组件  
蓝图中添加结点 `GetUserWidgetObject`, 再CastTo WB_DamageNumber  

+ 添加组件  
Pawn上可以添加结点 Add WCDamageText
