---
title: Ionic自动化构建之路
tags: 'Ionic, CI, Python'
date: 2017-03-04 15:51:33
---


### 前言
本文主要讲述``Ionic``在公司最新产品中的应用

### 技术背景
公司计划发布新的产品。为了利用现有的资源，快速构建产品，采用``ionic``作为基础框架。``ionic``是一款H5移动开发框架，打包插件基于``apache cordova``。

### 实践之路
实践过程:

* 安装``ionic``, ``cordova``框架。运行如下命令

```bash
npm install -y .
npm install -y ionic cordova
```

* 创建``ionic``项目。使用如下命令，可以指定包名

```bash
ionic start xxx(项目名称) --id xxx(包名)
```

* 下载需要的``cordova``插件
* 获取资源文件
* 更新配置文件
* 添加支持平台

```bash
ionic platform add android
```
* 构建项目

```bash
ionic build xxx(平台)
```

### 可持续集成
为了可持续集成，那么我这里将整个项目分为两部分，一部分固定不变的，即每次框架自动生成的文件。另一部分为我们一直需要修改的。因此这里的想法是，将改变的部分分离出去。每次运行脚本，自动生成文件。然后使用改变的部分覆盖自动生成的部分。这里改变的部分，主要包括配置文件，``Native``代码以及资源文件。替换完毕，打包签名出``apk``文件，并上传云服务。根据此思路，编写``python``脚本:

```python
# 覆盖配置文件
copy_file('config.xml', subfix)

# 覆盖build.gradle文件
copy_file('build.gradle', androidPath)

# 覆盖 AndroidManifest
copy_file('AndroidManifest.xml', androidPath)

# 覆盖资源文件
# 先删除资源文件
try:
    remove_tree(toDirectory + '/resources')
except OSError:
    pass

# 覆盖资源文件
copy_tree(fromDirectory, toDirectory + '/resources')

# 覆盖native代码
copy_tree(fromDirectory, toDirectory)
```

之后签名打包，上传云服务

### 难点

1. 由于默认初始化项目，会自动生成一个包名。为了指定包名，因此需要使用``--id``来指定包名
2. 自动生成的``config.xml``文件中包含了自动生成的包名，一旦打包签名时，会根据这个包名自动生成代码，因此需要将此自动包名，修改为需要的包名

### 总结
此项目目前能实现从项目初始化，构建到最后打包上传完全自动化。大大减少工作量。对于我个人也是一次从新框架，新CI(bitrise)到最终的自动化构建的一次完整系统的学习。

