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