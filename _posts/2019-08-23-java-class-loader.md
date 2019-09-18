---
layout:     post
title:      "复习07 Java中的类加载器"
subtitle:   ""
date:       2019-08-23 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


## 类加载过程

简单的类加载过程如下

1. 加载
2. 连接
   - 验证
   - 准备
   - 解析
3. 初始化


### 加载过程

通过全类名获取定义此类的二进制字节流，将字节流所代表的静态存储结构转换为方法区的运行时数据结构，在内存中生成一个代表该类的 Class 对象, 作为方法区这些数据的访问入口。

重写类加载器可以控制获取类的二进制流的方式，可以从自定义的地点导入类。

### 连接过程

1. 验证类文件的二进制流
   - 验证文件格式 编译代码的编译器版本是否符合当前虚拟机版本，魔数是否可接受、常量类型是否可处理
   - 验证元数据   对类文件描述符的验证，比如 final、abstract 等修饰符
   - 验证字节码   验证字节码中有没有语义错误
   - 验证符号引用

2. 准备解析
   准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。这时候进行内存分配的仅包括类变量（static），而不包括实例变量。这里所设置的初始值 "通常情况" 下是数据类型默认的零值（如 0、0L、null、false 等），除非加了 final，那么会赋予等号后的默认值

3. 解析
   将类常量池中的符号引用与内存中的地址关联起来，可以是对象指针或句柄，也可以是方法表中具体方法的偏移量

4. 初始化
   初始化阶段是执行类构造器 `<clinit>()` 方法的过程。对于初始化阶段，虚拟机严格规范了有且只有 5 中情况下，必须对类进行初始化：
   - 当遇到 new 、 getstatic、putstatic 或 invokestatic 这 4 条直接码指令时，比如 new 一个类，读取一个静态字段 (未被 final 修饰)、或调用一个类的静态方法时。
   - 使用 java.lang.reflect 包的方法对类进行反射调用时 ，如果类没初始化，需要触发其初始化。
   - 初始化一个类，如果其父类还未初始化，则先触发该父类的初始化。
   - 当虚拟机启动时，用户需要定义一个要执行的主类 (包含 main 方法的那个类)，虚拟机会先初始化这个类。
   - 当使用 JDK1.7 的动态动态语言时，如果一个 MethodHandle 实例的最后解析结构为 REF_getStatic、REF_putStatic、REF_invokeStatic、的方法句柄，并且这个句柄没有初始化，则需要先触发器初始化。


## 类加载器

所有的类都由类加载器加载，加载的作用就是将 .class 文件加载到内存。数组类型不通过类加载器创建，它由 Java 虚拟机直接创建。

- Bootstrap ClassLoader
  启动类加载器，一般由C++实现，是虚拟机的一部分。该类加载器主要职责是将JAVA_HOME路径下的\lib目录中能被虚拟机识别的类库(比如rt.jar)加载到虚拟机内存中。Java程序无法直接引用该类加载器

- Extension ClassLoader
  扩展类加载器，由Java实现，独立于虚拟机的外部。该类加载器主要职责将JAVA_HOME路径下的\lib\ext目录中的所有类库，开发者可直接使用扩展类加载器。该加载器是由sun.misc.Launcher$ExtClassLoader实现。

- Application ClassLoader
  应用程序类加载器，该加载器是由sun.misc.Launcher$AppClassLoader实现，该类加载器负责加载用户类路径上所指定的类库。开发者可通过ClassLoader.getSystemClassLoader()方法直接获取，故又称为系统类加载器。当应用程序没有自定义类加载器时，默认采用该类加载器。

每一个 ClassLoader 都拥有自己独立的元空间，类是由 ClassLoader 将其加载到 Java 虚拟机中，故类是由加载它的 ClassLoader 和该类本身一起确定其在 Java 运行时环境的唯一性。故只有同一个 ClassLoader 加载的同一个类，才能算是 Java 运行时环境中的相同的两个类。哪怕是来自同一个 Class 文件，即使被同一个虚拟机加载的两个类，只要 ClassLoader 不同，那么也属于不同的类。对于 equals()、isinstanceof() 等方法来判断对象的相等或所属关系都是需要基于同一个 ClassLoader。

### ClassLoader 类的源码

```java

protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 先查找已经加载的类
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 找不到的话先尝试父类加载，父类为null则用启动类加载器加载
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // 父类无法加载时调用findClass自己加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }

            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```


### 双亲委派模型

双亲委派是指加载器先尝试交给上层加载器处理，上层加载器不可处理时由自己处理，类似于反向的责任链设计模式。这样做的好处是不会出现同名的类由不同的加载器加载导致重复的情况。也保证了 java 的核心类由 `BootstrapClassLoader` 处理，不被别的加载器篡改。如果要覆盖这种模式，那么自定义一个类加载器，继承 `java.lang.ClassLoader` 然后重载 `loadClass` 即可。

ClassLoader类内部有一个parent属性，记录了这个加载器的父层级加载器是哪个，如果parent为null，那么会尝试用`BootstrapClassLoader` 加载。

### 应用场景

Tomcat、Android系统等自定义实现的类加载器，保证了Framework层和App层的分离，框架层的实现不会被修改替换，同时又可以资源共享，不用重复加载类。而应用程序实现层的类，互相隔离不会冲突。