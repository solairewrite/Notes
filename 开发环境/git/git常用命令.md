# git常用命令
## 目录
- [git常用命令](#git常用命令)
  - [目录](#目录)
  - [拉取](#拉取)
  - [git lfs 大文件存储](#git-lfs-大文件存储)
  - [忽略文件](#忽略文件)
  - [账号](#账号)
  - [远端](#远端)
  - [撤销本地修改](#撤销本地修改)

## 拉取
```
// 只拉取最近一次的提交
git pull --depth=1 --allow-unrelated-histories

// 拉取指定分支  
git clone -b mybranch url
```

## git lfs 大文件存储
```
// 查看 lfs 版本
git lfs -v

// 开启 lfs 功能
git lfs install

// 追踪文件
git lfs track *.uasset

// 查看当前追踪模式
git lfs track

// 查看当前追踪文件列表,在提交后可使用
`git lfs ls-files`  
```

## 忽略文件
```
// 忽略已经提交过的文件(.gitignore无效)
// 应用: 如每个人不同的配置文件,UE4的子地图,即使没有主动修改,也会因为材质变化而自动改动
git update-index --assume-unchanged 路径

// 查看已忽略的文件
git ls-files -v | grep '^h\ '

// 取消忽略
git update-index --no-assume-unchanged 路径
```

## 账号
```
// 查看账号邮箱
git config user.name
git config user.email

// 修改账号邮箱
git config --global user.name "solairewrite"
git config --global user.email "18817366719@163.com"

// 记住账号密码
git config credential.helper store
```

## 远端  
```
// 查看远端仓库地址
git remote -v

// 更改远端仓库地址
git remote set-url origin https://git.code.tencent.com/solairewrite/Faith.git
```

## 撤销本地修改
```
git reset --hard
```