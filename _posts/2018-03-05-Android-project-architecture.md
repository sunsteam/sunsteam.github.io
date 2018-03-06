---
layout:     post
title:      "Android 新项目构架"
subtitle:   ""
date:       2018-03-05 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---

> 基于 Kotlin，集成通用组件和基类的模版，依赖库的选择，辅助工具等。

<br>

### 1. 开发环境

Android Studio 3.0.1
Gradle 4.1

#### 1.1 Studio 开发效率插件

- `Android Styler` 快速抽取 Layout 中的属性到 Style
- `ButterKnife Zelezny` 生成 ButterKnife 代码
- `Android File Group`  按照命名规则对资源文件进行文件夹分组，类似于代码的分包，有利于快速找到通模块下的资源
- `Parcelable code generator` 在数据类中生成 Parcelable 接口的模版代码
- `GsonFormat` 根据 json 生成数据类，有 Kotlin 版本但不太完善
- `LayoutFormatter` 优化的 layout xml 代码格式化工具，符合开发习惯，默认的好看
- `MVP Helper` 生成 MVP Contract 模版代码，但是这个模版不太好用，需要自定义 （todo）
- `Selector Drawable Generator` 根据命名快速生成带状态的 drawable

---

### 2. 依赖库

目前先不抽取，等代码稳定之后考虑抽取组件，目前可以考虑抽取的部分包括：网络库封装、支付、升级、Utils

#### 2.1 官方扩展库

- `databinding` 可考虑开启，逻辑少的页面用 mvvm 会比较省代码，view 事件多而 model 事件少的情况也比较好使用
- `vectorDrawables` 以兼容模式运行，vectorDrawable 无需适配屏幕，因此只有图形形状时，比较好用
- `vectorAnimation` 可以实现一些矢量动画，看情况依赖
- `design` 如果项目有使用 behavior 等效果则开启，一般情况不需要，此库较大，而且只支持 5.0 + 效果。recyclerView，cardView 等应单独依赖。
- `multidex` 开启，Tinker 要用，可能有初始 Dex 加载问题，如果找不到类崩溃则需要 check。

#### 2.2 JSON 解析

- `fastjson` Alibaba 开源的库，它的 Api 我觉得比 Gson 简单，网上的对比显示性能也比 Gson 高一倍以上，因此首先考虑。与 Retrofit 、Gsonformat 也可配合。

#### 2.3 网络库

- `okHttp3` okHttp 不说了
- `logging-interceptor` okHtpp3 同 Group 下的这个拦截器可以实现网络访问的 log，用于 debug

<br>

- `Retrofit` 优点不说了，缺点：我觉得不灵活，必须要从对应 Service 开始请求，无法使用泛型封装到某个流程中进行动态变化的请求。如果谁有好的封装方法请告诉我。

<br>

- `OkGo` 另一个选择，同样是封装 okHttp3，支持 rx2，全中文文档，内部实现 cookie 缓存、请求结果缓存、封装了 Https 的创建 Api 、文件的上传下载等，可以自定义配置是否使用，okHttp Client 的创建部分参考了 Retrofit，Callback 部分则更灵活，原因在于其解析模型的获取（TypeToken) 不是由动态代理传递，而是需要自定义实现的。如果说缺点，我觉得是 rx 的支持部分，代码略丑，毕竟不是源生设计与 rx 合用的。

#### 2.4 图片库

- `Glide` 其他 2 个图片库我没用过，不过对比来讲，Picasso 是个简化版的 Glide，更少的代码量，但不支持 Gif，Fresco 和 Glide 差不多。Glide 方法数和包大小较大，但是支持功能更丰富，基本都实现了，对于包大小要求不严格的 app 来说，首选。可通过扩展库支持 okHttp 核心。

- `takePhoto` 拍照获取图片，保持图片和相册提取图片功能的实现，内部实现了 FileProvider、Permission 申请等操作，此库挺久没有维护了，而且作者开发的时候依赖了太多其他库，比如 Glide、rx1、easyPermission 等，随着这些库版本更新，可用性就下降，如果有其他好的选择，可以替换，否则就 fork 了改。

- `photoView` 图片缩放交互库

#### 2.5 响应式 rx

- `rxJava2` rxKotlin 还不熟
- `rxAndroid` 
- `rxlifecycle` 管理生命周期

其他的全家桶不准备用，rx 更新的时候，他们万一不更新，就有可能出问题。rxBinding 可以考虑。

#### 2.6 event 事件发布订阅

- `EventBus` 也可以自定义 rxBus

#### 2.7 Log

- `logutils` 可以接受所有类型的参数，1.4.0 版本是我用的，1.4.2 开始像 Logger 一样包裹了头尾，多了很多无用信息看着不舒服。
- `Timber` 支持 Log 的本地保存，有这个需求可以用

---

### 3. 基类封装和代码风格

封装几点原则：
- 封装不应该影响灵活性，如果封装完后这个类丧失了部分功能，那么就不应该封装。
- 使用继承的前提是：父类的封装是子类行为的基础，父类的属性子类也会有，也就是有垂直关系才用继承，如果是添加一个完全无关的功能，应该使用接口 + 装饰模式，Kotlin 中可以更方便的使用委托装饰。

代码风格
- 减少互相依赖，特别是一些常用页面或者逻辑类，类引用可能在项目各处散布，如果依赖更改就要到处找，应使用中介者模式进行一次封装，到时候只改中介者类就可以。
- 构造和重载应多使用享元模式，减少不变动的参数。
- 分包不能过细
- 简单页面不需要使用 mvp，额外增加了代码量，反而使用 MVVM 可以减少纯 get、set 代码
- 约束变量生命周期，如无必要，无需持有。
- 减少视图层级，能不嵌套，就不嵌套
- 检查重复绘制，减少不必要的 background

#### 3.1 列表

- 列表共通的封装部分主要是适配器部分的封装：

    **减少模版代码量** ： 

    - 基类中添加常用的 add 、remove 方法
    - 封装 getView 或 create/bindView 方法
    - 通过键值对数据结构缓存 findViewById

    **灵活的 viewType -> View**

    - 我只在 base 中对源生方法进行了一层封装，但网上有几篇博客是讨论这个问题，可以看看 [drakeet 大神的这篇博客][1]，略复杂。

    **处理数据和分页**

    - 这部分可以对数据转型处理操作进行抽象，然后子类以不同方式数据渠道实现数据提取，比如：从文件，从网络，从数据库。
    
    - 分页操作都是一样的，可以抽象。

    - 适配器状态可以抽象，一般分为 无数据、可能有更多数据、已获取全部数据、获取数据中、内部错误、数据源错误

    **下拉刷新和上拉加载**

    - 参考开源库的实现方式，一般有的方式为 
        1. 桥接模式 自定义组合控件
        2. 装饰模式 进行包装后加载，其实也是组合控件，只是过程动态

- `ListView` ListView 该有的都有，普通列表使用比 RecyclerView 更简单，完全放弃他是不合适的，应该看场景。

- `RecyclerView` 与 ListView 相反， RcyclerView 的设计真正做到了完全解耦，非常灵活。在有自定义的动画，视图或者方向布局的时候使用它。

- 但是太灵活使用时候就要多配置东西，所以我们也把 RecyclerView 封装成 ListView 的样子。。。`BaseRecyclerViewAdapterHelper` 一个用户量很大的 RecyclerView 封装库，功能很全，用户量巨大，中文文档，开发团队维护很勤劳。团队还抢到了这个域名 ：[www.recyclerview.org](http://www.recyclerview.org/)

#### 3.2 页面

todo

#### 3.3 MVP

todo

#### 3.4 View

todo

---

### 4. 工程化工具

#### 4.1 屏幕适配

参考 [AdaptDpToScreen][2]

#### 4.2 代码安全

- 混淆，使用模版代码，更换字典
- 资源混淆，使用 AndResguard
- 加固，使用腾讯的，与 Tinker 的兼容性更好
- 保护签名文件，不应放于 Project 中
- 用 SO 保存字符串常量 （考虑中，未实施验证）

#### 4.3 Bug 收集 + 热更新

- Bugly + Tinker ， 参考 [GradleOptimize][3]

#### 4.4 用户行为收集和自定义埋点

- 参考 [UserLogger][4]

#### 4.5 测试

- 书我倒是买了，一直没有翻啊。 思路 MVP + Dagger2 + Mock 可以比较方便的做单元测试，但这方面没什么研究

#### 4.6 多渠道打包

- 没什么研究，但是方法很多应该不难。

#### 4.7 持续集成

- Jenkins 的部署 （已基本研究完成），可以参考 [使用 Docker 构建持续集成与自动部署的 Docker 集群][5]

#### 4.8 多人开发

- MVP 模式 作为基础支持
- Git 的使用习惯
    - develop 分支做多人分支 Merge，Master 做发布
    - bug fix 另开 branch ，测试完成合并到自己的分支
    - 实验性功能另开 branch
- 组件化分 moudle 开发

#### 4.9 任务管理工具

- [Leangoo][6] 免费的看板管理工具，支持多人合作，任务池（可分配，可领取），任务量统计、任务时间管理

---

### 5. 新技术

#### 5.1 组件化和插件化

需要考虑的问题：
- 实现方式是否和 Tinker 热更新功能有冲突
- 业务之间是否能完全解耦
- 项目大小，需要分解的程度

目前可选方案包括下面 3 个，解耦程度依次递增，详细情况还需学习并做 Demo

- `Arouter` 阿里巴巴
- `VirtualApk` 滴滴
- `RePlugin` 360

#### 5.2 依赖注入

- Dragger2 的学习，目前正在看，这部分风险较小，可以优先考虑加入。


[1]: http://drakeet.me/multitype/
[2]: https://github.com/sunsteam/AdaptDpToScreen
[3]: https://github.com/sunsteam/GradleOptimize
[4]: https://github.com/sunsteam/UserLogger
[5]: http://www.cnblogs.com/duyinqiang/p/5696254.html
[6]: https://www.leangoo.com/
