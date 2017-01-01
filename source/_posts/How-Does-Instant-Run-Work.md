---
title: How Does Instant Run Work
tags: Android
date: 2016-10-14 15:04:05
---


[原文](https://medium.com/google-developers/instant-run-how-does-it-work-294a1633367f#.89kk542oe)

``Instant Run``，``Android Studio``的一项神奇的功能，用于减少增量代码的构建和部署时间。它看起来很神奇。你首次运行或者调试，你就如你预期般的那样工作，之后每次代码的改动，会花很少的时间去构建和部署

#### 原理

构建图

![instant build](./instant_run_build.png) 


``Instant Run``的目标很简单

> ***移除尽可能多的步骤，使剩下的东西尽可能快***

具体是:

* 只构建和部署新增的东西
* 不重新安装应用
* 不重新启动应用
* 不重新启动``Activity``

#### 热部署，温部署，冷部署

![hot_warm_cold](./hot_warm_cold.png)

**热部署**: 部署新的改变，不需要重新应用，甚至不需要重启当前``Activity``。能用于方法内简单的改变

**温部署**: ``Activity``需要重启后，新的改变才能生效。通常用于资源改变

**冷部署**: 应用重启，但不重新安装。任何结构型变化，如继承关系或者方法签名会使用冷部署

#### 打包流程

![merged_combined](./merged_combined.png)

``manifest``文件被合并，打包。伴随着资源一起打包进``APK``。``java``源代码被编译成``字节码``，转换成``.dex``文件。它们也会打包进入``APK``

**首次点击运行或调试(``Instant Run``打开), ``Gradle``做的额外操作**

![first_run_instant_run](./first_run_instant_run.png) 

字节码被添加到``.class``文件中，一个新的``App Server``类被注入到``app``中

一个新的``Application``类定义也被加入到``App``，注入自定义类加载器以及将启动``App Server``。一般来说，``manifest``会被修改以便``app``能使用(如果你创建了自己的``Application``类，``Instant Run``版本会代理这个``Application``类)

这时``Instant Run``运行了。因此如果你改变了代码，``Instant Run``会尝试避免使用热，温，冷部署来避免全构建

>应用``Instant Run``改变之前，``as``会检查``Instant Run``内是否有一个打开的``Socket``连接着``App Server``

#### 热部署

![hot_swapping](./hot_swapping.png)

``as``监控开发过程中哪些字段被修改了。运行自定义``Gradle``任务来未修改的类生成``.dex``文件

这些新``.dex``文件被``as``挑选，并部署到``app``中的``App Server``

由于类的原始版本已经存在了，``Gradle``转换更新版本以便高效覆盖这些之前存在的类。转换结束，更新的类被``App Server``使用自定义的类加载器加载。

从现在起，每次方法被调用，这个注入到原始类文件中的监控类都会检查是否已经更新。如果更新了，那么后续操作会代理到新的覆盖类中。

#### 温部署

温部署重启``Activity``。当``Activity``启动，资源被加载。因此更新资源需要重启``Acitivty``以便强制资源重新加载

当前，任何资源的改变会导致，所有的资源被重新打包到``app``，但使用增量打包器会只打包和部署更改的资源

>注意温部署对于``manifest``的改变是无效的，因为``manifest``信息的读取实在``apk``安装的时候。``manifest``的改变会触发全量构建和部署


#### 冷部署
部署后，``app``和其子项目被分成10个片段，每个片段有自己的``dex``文件。类安装它们的包名分割。使用冷部署，修改类会要求所有其他在相同片段中的类重新加载。

这个策略依赖于运行时加载多个``dex``文件的能力。``5.0``以上采用``ART``具备这种能力。之下会采用全量构建部署

#### ``Instant Run``技巧和提示

* ``manifest``的修改会导致全量构建部署
* ``Instant Run``值监控主进程，如果``app``使用多进程。热部署和温部署在其他进程中会降到冷部署，如果低于5.0，会采用全量构建部署
* ``Windows``防火墙可能会导致``Instant Run``无法启动
* ``Instant Run``不支持``Jack``编译器

