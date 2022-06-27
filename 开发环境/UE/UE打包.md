# UE打包
## 目录
- [UE打包](#ue打包)
  - [目录](#目录)
  - [添加引用资源](#添加引用资源)
    - [添加DataAsset](#添加dataasset)
    - [添加lua文件夹](#添加lua文件夹)

## 添加引用资源
### 添加DataAsset  
新创建一个PrimaryAssetLabel,它继承自PrimaryDataAsset  
将你的DataAsset加到ExplicitAssets数组中  
Project Settings -> Game -> Asset manager  
Primary Asset Types To Scan 应该自动加上PrimaryAssetLabel  
Primary Asset Rules 中手动加上刚刚创建的PrimaryAssetLabel  

### 添加lua文件夹
lua文件夹在Content/Script下  
Project Settings -> Project -> Packaging  

Additional Asset Directories To Cook 数组加上(要手动添加的)蓝图所在的文件夹,如 /Game/Config  
Additional Non-Asset Directories to Cook 数组加上 /Game/Config  