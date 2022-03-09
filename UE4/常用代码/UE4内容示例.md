# UE4内容示例
## 关卡
| 序号 | 关卡 | 描述 | 是否学习 |
| :-: | :- | :- | :-: |
| 1 | [Animation](#Animation) | 动画 | & |
| 2 | Audio | 声音 | mark |
| 3 | [Blueprint_Communication](#Blueprint_Communication) | 蓝图之间的交互 | & |
| 4 | [Blueprint_HUD](#Blueprint_HUD) | 简单的HUD | & |
| 5 | Blueprint_Input_Examples | 简单的输入控制小游戏 |  |
| 6 | Blueprint_Mouse_Interaction | 鼠标选中小球 | & |
| 7 | Blueprint_Splines | 曲线 |  |
| 8 | BlueprintRenderToTarget | 渲染到贴图,流体表面模拟 |  |
| 9 | Blueprints_Advanced | 简单的蓝图技巧示例 |  |
| 10 | Blueprints_Overview | 基础 |  |
| 11 | Character_Rendering | 角色渲染 |  |
| 12 | Cloth | 布料 | & |
| 13 | [Decal](#Decal) | 贴花 | & |
| 14 | DynamicSceneShadows | 动态阴影 |  |
| 15 | [Effects](#Effects) | 粒子特效 | & |
| 16 | ExampleProjectWelcome | 欢迎地图 |  |
| 17 | FBX_Import_Options | FBX导入,动画相关 |  |
| 18 | Geometry_Editing | BSP画刷创建简单的关卡地形 |  |
| 19 | HoudiniNiagara | 霓虹灯特效 |  |
| 20 | Landscapes | 创建地形 |  |
| 21 | Leveldesign_Workflow | 关卡完善流程 |  |
| 22 | Level_Scripting | 关卡蓝图 | & |
| 23 | Level_Streaming | 动态加载关卡 | & |
| 24 | Lighting | 光照 |  |
| 25 | Material_Advanced | 高级材质 |  |
| 26 | Material_Example | 材质 |  |
| 27 | Material_Instances | 材质实例 |  |
| 28 | Material_Nodes | 材质节点 |  |
| 29 | Material_Properties | 材质属性 |  |
| 30 | Math_Hall | 数学 |  |
| 31 | MorphTargets | 形变目标 | & |
| 32 | NavMesh | 导航 | mark |
| 33 | Network_Features | 网络同步 |  |
| 34 | Niagara | 新一代粒子特效 | mark |
| 35 | Paper2D | 2D |  |
| 36 | ParallaxOcclusionMapping | 渲染相关 | 略 |
| 37 | Particles_Intro | 粒子特效 | & |
| 38 | PhysicalAnimation | 物理动画 |  |
| 39 | Physics | 物理 |  |
| 40 | PivotPainter | 轴向绘制 | & |
| 41 | PivotPainter2 | 轴向绘制 | 略 |
| 42 | PostProcessing | 后期处理 | mark |
| 43 | ProceduralMesh | 过程材质 |  |
| 44 | Reflections | 反射,镜面效果 |  |
| 45 | SkinRendering | 皮肤渲染 |  |
| 46 | StaticMeshed | 静态网格 |  |
| 47 | UMG | UI | & |
| 48 | Volumes | 体积 | 略 |
___
### Animation
+ Character BP  
  
  通过输入控制 Direction, Speed 等变量  

+ Animation BP  
  
  + 事件图表  
  
    获取Character的变量,并设置 AnimBP 内的变量(e.g. Direction, Speed)  

    Blueprint Update Animation 事件每帧执行  
    Try Get Pawn Owner / Get Owning Actor 节点获取动画所有者  
    Cast to MyPawnBP 转换为自己的角色类,获取属性值

  + 动画图表  

    添加状态机,各种动画节点,混合节点,返回输出姿势


#### Animation Mode
播放动画的模式 SkeletalMeshActor -> Animation Mode  
+ Use Animation Blueprint 使用动画蓝图  
+ Use Animation Asset 使用指定动画  
#### Locomotion BlendSpace
通过 Direction(-180, 180), Speed(0, MaxSpeed) 二元混合移动动画  
Idle  
Walk_Fwd, Walk_Left, Walk_Right, Walk_Bwd  
Run_Fwd, Run_Left, Run_Right, Run_Bwd  
#### Aim Offset
通过 Yaw(-180, 180), Pitch(-90, 90) 二元混合瞄准动画  
下面的动画命名: AO_YawPitch  
Yaw 方向: C, LB, L, RB, R  
Pitch 仰角: C, D, U  
总共 5*3=15 个  
AO_CC, AO_CD, AO_CU  
AO_LBC, AO_LBD, AO_LBU  
...  
#### Montage
BP 添加节点 Skeletal Mesh(组件引用) -> Get Anim Instance -> Montage Play  
AnimBP Anim Graph 添加节点 Slot  
#### 拖尾特效
Skeletal Mesh 上添加PSC  
BP 中添加节点 PSC(组件引用) -> Begin Trails  
___
### Blueprint_Communication
蓝图之间交互的3种方式:  
+ Custom Event  
  
  A在蓝图中维护对B的引用,调用B的 Custom Event  

+ Event Dispatcher  
  
  将 A 的 Event Dispatcher 绑定到 B 的 Custom Event  
  然后 A 调用 A 自己的 Event Dispatcher, 就能触发 B 的 Custom Event  

  绑定可已在 Level BP 或 A,B 自己的 BP 中进行  

+ Interface  
  
  声明接口类 BPI, 声明函数 Func()  
  A 直接调用 Actor 的接口函数 Func()  
  B 类设置中实现接口 BPI, 并实现函数 Func() 
___
### Blueprint_HUD
GameMode -> HUD Class 设置的 HUD 会直接显示  
Receive Draw HUD 事件每帧执行,提供屏幕X,Y尺寸,用于绘制HUD  
___
### Decal
DecalActor,向在体积内的StaticMesh投影  
修改DecalMaterial属性控制贴花样式,可以为动画贴花  
StaticMesh -> Rendering -> Receives Decals: 是否接受贴花  
Sort Order控制贴花绘制顺序  
___
### Effects
#### 1.2 速度锥模块
#### 1.3 发射器初始位置
Emitter Initial Location 模块  
粒子从其它粒子的位置生成  
#### 1.4 重力
在空间中指定一个点,所有粒子都被吸引到该点  
#### 1.5 碰撞
Collision(Scene Depth)模块  
#### 1.7 骨架网格物体的骨骼的粒子发射
通过骨骼/插槽模块实现  
#### 1.8 半透明粒子阴影投射
粒子勾选 Cast Shadow  
光照勾选 Cast Translucent Shadow  
#### 2.3 黑体节点
黑体辐射,温度控制颜色  
#### 2.4 Event Generator 事件生成器
使用粒子生命周期中可能发生的各种类型的事件,产生,小王,碰撞,爆炸等  
烟花: 用粒子的消亡创建其它粒子  
#### 2.5 Ribbon TypeData 从其它粒子流送的条带轨迹
2个粒子发射器,一个发射器发射源粒子,产生条带  
一个发射器有Ribbon TypeData模块和Source模块,用于指定源发射器
#### 2.6 体积粒子效果的光照半透明度
#### 2.7 Light 光源
粒子发光
#### 2.8 Noise 噪点
Beam TypeData: 光束类型数据,创建连接两个点的效果,如子弹轨迹  
Beam 借助 Noise 模块产生电弧  
