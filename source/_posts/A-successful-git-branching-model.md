---
title: 优秀的git分支模型
date: 2016-04-28 16:00:43
tags: git
---


### 好好学习git以及git-flow

![](./git-model.png)


### 分散又集中

我们的``git``模型中有个中央仓库"truth"。注意这个仓库只能被当做中央仓库(虽然技术层面上讲，``git``里并没有中央仓库这个概念)。我们把这个仓库当做``origin``，因为所有的``git``用户都对这个名字比较熟悉。

![](./centr-decentr.png)

每个开发者从``origin``拉取或者推送代码。这种关系背后，每个开发者也可以从其他子团队的节点拉取改变的东西。对于多人协作开发时这可能非常有用。上图中有Alice和Bob, Alice和David，Clair和David多个子团队。

技术层面上讲，就是Alice定义了一个远程Git，名叫bob,指向Bob的仓库。

### 主分支

我们的开发模型受现有模型的启发。中央库有两个一直存在的主分支：

* master
* develop

每个``git``用户应该都对``origin``上的``master``主分支很熟悉。与``master``并行的另一个分支叫``develop``.

我们认为``origin/master``的``HEAD``的代码总是保持着产品状态。

我们认为``origin/develop``的``HEAD``总是保持着发布之前的所有功能。我们称着叫"合成分支"

当``develop``的代码已经稳定并准备发布的话，这时所有的修改需要合并到``master``并打上``release``号。这会在之后更深入的讨论。

因此，每次有新的修改合并到``master``时，默认就有一个新的产品发布了。

### 支援分支

除了主分支``master``和``develop``之外，我们的开发模型使用了各种支援模型来在成员之间并行开发如,新功能分支，产品准备发布分支以及紧急修复分支。不像主分支，这些分支的生命周期都是有限的，因为它们最终都会被移除。

我们使用的分支类型：

* 功能分支
* 发布分支
* 紧急修复分支

#### 功能分支
<img align="right" width = "100" height = "300" src="./fb.png" >

这个分支必须从``develop``分出，最后必须合并到``develop``，命名规则：除了``master,develop,release-*, 或者hotfix-*``之外

功能分支被是用来开发新功能。当开始开发新功能时，这个功能的最终发布节点可能是不知道的。功能分支的重要性在于它与开发功能的周期一样，但最终会合并到``develop``或者会被抛弃。功能分支只存在于开发仓库，不在``origin``。

<p style="text-align: center;">创建功能分支</p>

```
$ git checkout -b myfeature develop
Switched to a new branch "myfeature"
```

<p style="text-align: center;">合并一个完成的功能</p>

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff myfeature
Updating ea1b82a..05e9557
(Summary of changes)
$ git branch -d myfeature
Deleted branch myfeature (was 05e9557).
$ git push origin develop
```

``--no-ff``标记合并的时候回创建新的提交，虽然合并能够使用``fast-forward``。这样做能够清晰的看到这个分支的所有提交。比较如下

![](./merge-without-ff.png)

通过左图可以看出，当你想把这个功能去掉时会很方便找到功能的开始节点。

#### 发布分支

从``develop``分出，合并回``develop``和``master``，命名规则:``release-*``

新产品要发布的话可以使用发布分支。它允许修复一下小的``bug``以及准备元数据(版本号，构建日期等)。

关键点在于必须合并完所有需要的功能分支后才能开出发布分支。


<p style="text-align: center;">创建发布分支</p>

```
$ git checkout -b release-1.2 develop
Switched to a new branch "release-1.2"
$ ./bump-version.sh 1.2
Files modified successfully, version bumped to 1.2.
$ git commit -a -m "Bumped version number to 1.2"
[release-1.2 74d9424] Bumped version number to 1.2
1 files changed, 1 insertions(+), 1 deletions(-)
```

创建分支后，写入版本号。在这,``bump-version.sh``是一个能修改版本号的脚本，然后提交写入的版本号。

可以在发布分支上修改``bug``。但绝对禁止在这个分支上加大的功能。它们必须合并到``develop``，因此只能等到下一次大的发布。

<p style="text-align: center;">结束发布分支</p>

当发布分支已经准备好，需要做一些操作。首先把发布分支合并回``master``。下一步，这些提交需要标记一下以便后续查找。最后发布分支需要合并到``develop``以便将来的发布版本也能包含当前发布版本的``bug``修复。

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2
```

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
```

这可能会导致冲突，没关系，解决它就好了。现在我们可以删除发布分支了

```
$ git branch -d release-1.2
Deleted branch release-1.2 (was ff452fe).
```

#### 紧急修复分支

<img align="right" width = "200" height = "300" src="./hotfix-branches.png" >

可能从``master``开出，必须合并回``develop``和``master``，命名规则：``hotfix-*``

紧急修复分支很像发布分支，因为它是另一个即将发布的产品。它主要解决一下比较严重的``bug``。紧急分支必须从``master``相应产品的标记分出。它的作用在于其他人可以继续别的工作而不受修改``bug``人的影响。

</br>
</br>
</br>
</br>

</br>


<p style="text-align: center;">创建紧急修复分支</p>

```
$ git checkout -b hotfix-1.2.1 master
Switched to a new branch "hotfix-1.2.1"
$ ./bump-version.sh 1.2.1
Files modified successfully, version bumped to 1.2.1.
$ git commit -a -m "Bumped version number to 1.2.1"
[hotfix-1.2.1 41e61bb] Bumped version number to 1.2.1
1 files changed, 1 insertions(+), 1 deletions(-)
```

别忘了写入版本号！

#### 结束紧急修复分支
当分支结束时，需要合并到``master``和``develop``以便之后的版本能够包含这些修复。这跟发布分支的做法一致。

首先，更新``master``并标记``release``

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2.1
```

然后，合并分支进入``develop``

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
```

有种情况需要知道，当发布分支当前已经存在的话，紧急修复分支需要合并到当前发布分支中，而不是``develop``。最终发布分支中的紧急修复代码能够合并到``develop``。

最后，删除临时分支：

```
$ git branch -d hotfix-1.2.1
Deleted branch hotfix-1.2.1 (was abbe5d6).
```

### 总结
这个分支模型没有什么新的东西，本文中的大图在我们的项目中非常有用。它是一种优雅的开发模型，能非常简单的进行团队协作。

[原文](http://nvie.com/posts/a-successful-git-branching-model/#decentralized-but-centralized)