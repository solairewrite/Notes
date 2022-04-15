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
