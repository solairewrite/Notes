# World Machine
创建地形的软件  

+ 设置地图尺寸  
World Extents and Resolution  
可以在UE4中新建关卡,在Modes->Landscape中查看尺寸,这里是512*512  
设置高度2000m  

+ Device View  
在这里添加结点  

+ 创建随机地形  
Devices->Generators->Advanced Perlin  

+ 地形分层  
Devices->Filters->Terrace  

+ 自然腐蚀  
Natural->Erosion  

+ 3D View 中预览添加节点后的效果  

+ 保存,导出  
Devices->Outputs->Height Output  
设置导出路径,设置文件格式RAW16,点击Write output to disk  

+ UE4导入  
新Level中Modes->Landscape,import from file  
选择创建好的材质MI_Landscape  
ScaleZ,这里设置了400  
Number of Component 设置了 2*2,太多会导致边缘拉伸成平的  

没有颜色,需要在Paint中,Layers点击'+',创建Weight-Blended Layer  
