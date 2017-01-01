---
title: Android开发30条经验
date: 2016-03-14 15:16:24
tags: Android
---

## Android开发30条经验

有两种人，一种是自己摸索来学习，一种是通过别人的建议。这儿有些经验我想与你们分享：  
<!--more-->
1. 使用任何第三方库之前，请慎重考虑，一会是一个很**严重**的提交。
2. 如果用户看不到它，**别绘制它**。
3. 除非你真的需要使用数据库，否则别用。
4. 方法数很容易超过65K，**```multidexing```**可以帮助你。
5. ```RxJava```是比```AsyncTasks```更好的选择。
6. ```Retrofit```是最好的网络库。
7. 使用```Retrolambda```来简化代码。
8. 混合使用```RxJava, Retrofit, Retrolambda```很棒！
9. ```EventBus```很棒，但不要使用太多，它会让代码很难调试。
10. [按功能分包，而不是按层分包](https://medium.com/the-engineering-team/package-by-features-not-layers-2d076df1964d)。
11. 把任何东西都从主线程移除掉。
12. 使用```lint```来帮助你优化布局。
13. 如果你使用```gradle```，尽一切[可能](https://medium.com/the-engineering-team/speeding-up-gradle-builds-619c442113cb)来加速。[方式](https://medium.com/@shelajev/6-tips-to-speed-up-your-gradle-build-3d98791d3df9#.60bijkglh)
14. [提取编译报告](https://medium.com/the-engineering-team/speeding-up-gradle-builds-619c442113cb)，看看什么比较耗时。
15. 使用[知名](http://fernandocejas.com/2015/07/18/architecting-android-the-evolution/)的架构
16. 测试很花时间，但它使代码更健壮。
17. 使用依赖注入使项目更具模块化和更方便测试。
18. 听听[fragmentedpodcast](http://fragmentedpodcast.com/)将会对你很有帮助。
19. 不要使用你自己的邮箱作为android市场的发布账号。
20. 总是使用合理的输入类型(EditText)
21. 使用**analytics**来查找使用的模式和隔离代码。
22. 总是更新到最新的库(使用```dryrun```来测试它们)
23. 你的服务应该做它们需要做的事情，尽可能快的销毁。
24. 使用```Account Manager```来建议登录用户名和邮箱地址。
25. 使用可持续集成平台来构建和发布你的产品。
26. 不要运行自己的```CI```服务器，维护这个服务器是很耗时的。使用```circleci, travis或者shippable```，它们很便宜，并且需要关注的东西很少。
27. [自动部署到市场](https://github.com/Triple-T/gradle-play-publisher)。
28. 如果一个库很巨大，你仅仅只是需要当中的一小部分功能，简化它（使用proguard)。
29. 别使用太多的模块。如果模块不是经常修改，应该考虑将其打包成```.jar/.aar```，这样构建速度会大大提高。
30. 使用SVG
31. 使库抽象化，这样能很方便的切换到别的库。
32. 监控连接和连接类型。
33. 监控电量和电池
34. 用户界面就像小丑，如果你需要解释它，那还不如不要。
35. 性能测试很重要。


[原文](https://medium.com/@cesarmcferreira/building-android-apps-30-things-that-experience-made-me-learn-the-hard-way-313680430bf9#.1mrpcgfuo)