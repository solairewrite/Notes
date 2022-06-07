# UE配置
## 目录
- [UE配置](#ue配置)
    - [目录](#目录)
    - [禁用代码优化](#禁用代码优化)
    - [自动打开蓝图](#自动打开蓝图)
    - [配置Idea](#配置idea)
    - [进程](#进程)
    - [加速setup.bat下载速度](#加速setupbat下载速度)
    - [每次编译都会执行 using git status 占用大量时间](#每次编译都会执行-using-git-status-占用大量时间)
    - [AI调试小键盘](#ai调试小键盘)
    - [编译版本](#编译版本)
    - [UE5主题](#ue5主题)
    - [UE5自动打开上一个项目](#ue5自动打开上一个项目)

## 禁用代码优化
```
// .build.cs
OptimizeCode = CodeOptimization.InShippingBuildsOnly;
```

## 自动打开蓝图
Editor Preferences -> Restore Open Asset Tabs On Restart  

## 配置Idea
Editor Preferences -> Source Code Editor  

打开新窗口位置: Asset Editor Open Location  

## 进程
Editor Preferences -> Run Under One Process  

## 加速setup.bat下载速度
编辑setup.bat  
```
set PROMPT_ARGUMENT=--prompt --threads=20 --exclude=Linux --exclude=HTML5 --exclude=IOS --exclude=Mac --exclude=Android  --exclude=osx32 --exclude=osx64
```

## 每次编译都会执行 using git status 占用大量时间
修改 \Engine\Saved\UnrealBuildTool\BuildConfiguration.xml

```
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
    <SourceFileWorkingSet>
        <Provider>None</Provider> 
        <RepositoryPath></RepositoryPath> 
        <GitPath></GitPath> 
    </SourceFileWorkingSet>
</Configuration>

```

## AI调试小键盘
Project Settings -> (Engine)Gameplay Debugger -> Input  

## 编译版本
如果源码版引擎编译成功上传后,发布版无法打开项目  
解决方式:  
1. 使用发布版的Engine/Build/Build.Version文件,替换源码版的  
   此时重新编译项目的话,应该会导致引擎重新编译  

2. 项目Binaries/Win64文件夹下的.target, .modules等也要上传  
   关注里面的BuildId字段

3. 可能引擎要使用Development Editor,这点不确定  

## UE5主题
UnrealEngine\Engine\Content\Slate\Themes 文件夹下添加XXX.json主题文件  
在Editor Preferences -> Theme 中选择主题  

## UE5自动打开上一个项目
File -> Open Project -> Always load last project on startup  
