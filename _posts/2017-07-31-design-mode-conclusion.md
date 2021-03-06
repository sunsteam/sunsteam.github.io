---
layout:     post
title:      "设计模式小结"
subtitle:   ""
date:       2017-07-31 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - common
---

设计模式是为了代码易于阅读，易于复用，易于扩展而总结出的规律。

### 面向对象的五原则（SOLID ）

为了使代码复用，在一个类内部应尽量有单一功能的完整实现（内聚），减少对外部具体实现的依赖（耦合）。

在设计代码时，应考虑到以下原则：

- 单一职责： 一个类应专注负责某一个功能的实现。

- 开放 / 封闭原则： 对扩展开放， 对修改封闭。可以通过增加新的实现来应对变化，尽量减少对已有实现的改动。

- 接口隔离原则： 多个专用接口比一个通用的接口好。一个类不应实现用不到的接口。 换句话说， 不要过度封装。

- 里氏替换原则： 抽象应该可以被具体替换而不影响正常运行。

- 依赖倒转原则： 抽象不依赖细节，细节依赖抽象。代码应依赖于抽象，通过具体实现来扩展。


### 单例模式

用途： 减少不必要的对象创建。

场景： 各种全局对象或者隐藏创建方式的对象。

### 代理模式

用途： 保留一个实现功能的对象，只暴露应该暴露的方法。在代理方法执行的前后执行其他的操作。

#### java 的动态代理

用途： 委托类需要实现一个接口。动态生成一个类作为代理类和委托类之间的桥梁。 通过动态生成的类在接口方法的调用前后执行代码。用于实现切面编程。

场景： Retrofit 的 Service 生成。


### 装饰模式

用途： 为一个类进行包装，添加其他功能， 又不影响其核心功能。这种添加组合是不确定，可以一直包装下去。 装饰模式的装饰类应与被装饰类来自同一个父类或接口，这样才能保证核心功能的实现，并能够被其他装饰类接收。

场景： BufferedReader 包装并添加了 readLine 方法。


### 建造者模式

用途： 对实例构建过程的封装。按照固定的步骤构建复杂对象，但其中每道步骤是可以不同的，可以使用相同的流程构建出不同的实例。

场景： 构建一个有特定部件的对象。比如 AlertDialog，由标题、提示语、按钮文字等固定部分组成。

### 观察者模式

用途: 将代码之间的关联关系作为事件，解耦事件的发布者和事件的接受者。两者之间基于事件实现可以一对多或多对多的交互而不耦合。

场景： 观察者模式在 Android 的实现可以分为 4 类： reactX， 接口回调， 总线， 广播。 是最好用的设计模式。

### 外观模式

用途: 隐藏子系统之间的复杂关联而抽象出一个通用接口。常用与功能模块之间的分层设计。

场景： 比如数据获取层可能由多个子系统分级读取： 内存缓存 > 本地缓存 > 网络, 获取到数据后还要分级存入。抽取一个通用的 get/set 方法，隐藏分级读取、存入的子系统，就是外观模式。

### 适配器模式

用途： 对转换的抽象。

场景： AdapterView 通过一个适配器将数据模型转换成 View， 并将转换的部分抽象为 Adapter，以此实现解耦。
      Retrofit 中对 Call 的转换也属于适配器模式。


### 工厂模式

用途： 创建一个创建对象的工厂，将创建实例的方式封装为成为可扩展的子类。

场景： 各种创建对象的地方，并且需要考虑将来可能的扩展。

### 抽象工厂模式

用途： 和工厂方式一样，区别是不是创建单一的实例，而是创建一系列相关的实例。


### 职责链模式

用途： 一种多层次处理事件解耦的方法。低层的处理者只关心传递给所对应的上层处理者而不需要知道整个层次的处理方式。

场景： 需要多层次依序处理事件的场景，比如 View 树的点击事件处理。

### 中介者模式

用途： 迪米特法则的扩展，将依赖集中处理和分配， 减少代码中耦合的数量（转移为依赖中介者）。使用过度可能导致中介者肿胀

场景： 页面路由功能。

### 享元模式

用途： 封装一部分共有特性为内部状态，动态变化的部分作为外部状态传入，减少需要创建的对象数量，提高复用性。

场景： 各种可以抽取封装的对象。

### 命令模式

用途： 命令需要一个下指令的调用者和一个执行命令的执行者，命令模式是对执行请求的封装。方便被调用者方管理，一般对请求增加记录，取消这样的操作。

场景： 包含请求的场景，比如获取数据，获取图片。 OKhttp 的 Call 使用命令模式。

### 解释器模式

用途： 定义一个语法规则，相当于使用一个缩写代替了复杂的内容。

场景： 简化代码的时候。各种 annotation，apt 编程库，如 ButterKnife。使用一个注解就生成了模版代码。

### 策略模式

用途： 策略是对算法的封装。方便对实现方式的扩展。

场景： 达到目的有不同算法的时候，或者算法可能扩展的时候。比如计算积分使用百分比还是使用等级乘以在线时间长。


### 组合模式

用途： 一系列对象有类似树的层次结构，且有共有特性可以被操作。将添加和移除子项的部分以及共有特性抽取出来。

场景： View 和 ViewGroup 树


### 访问者模式

用途: 在数据结构稳定的情况下，对结构的操作抽象为访问者。方便在不影响结构的情况下扩展新的操作。

场景：没有找到实际使用的场景。


### 备忘录模式

用途： 在合适的时机保存 / 读取对象的状态。

场景： 用的不多，涉及到销毁，重建时需要考虑的。参考 Activity 的 onSaveState 和 onRestoreState， Preference 等。

### 状态模式

用途: 对不同状态下行为的封装。

场景：状态较多或行为较复杂，导致判断分支代码阅读性差的情况，或者是状态可能会扩展的情况。

### 桥接模式

用途：将复杂对象内的功能拆分成独立的模块，好处是模块在扩展的时候可以分别继承。增加内聚，降低耦合。

场景： 对象内有可以独立分类的部分，并且该部分可能扩展。

### 模版方法模式

用途： 抽取模版代码。

场景： 平时都有用到。

### 迭代器模式

用途： 抽象判断是否有下一项并取出的操作，根据数据结构的不同进行封装。

场景： 一般通用数据结构实现好了，不用太关注。

### 原型模式

用途： 复制对象的属性来生成一个克隆对象，不需要经过创建过程。

场景： 也比较少，对象生成过程较复杂，并且属性支持克隆时。
