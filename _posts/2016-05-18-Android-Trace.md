---
layout:     post
title:      "Android 性能优化实例：通过 TraceView 定位卡顿问题"
subtitle:   ""
date:       2016-05-18 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - Advance
---

## 背景

项目中使用了鸿洋大神的TreeView树状结构控件， 但是由于在主线程中使用了注解/反射来定位节点， 内容一多就有点卡顿。因此通过android device monitor中的性能分析工具定位并优化这一问题并记录过程。

[树状结构控件看这里](http://blog.csdn.net/lmj623565791/article/details/40212367)

其他优化内容可以看 [Android性能优化](http://blog.csdn.net/sunsteam/article/details/50612474)

## 准备工作

1. 打开tools菜单 —— android——android device monitor
2. 选中要追踪的app进程——点击Start Method Profiling
3. 操作想要追踪问题的界面或行为使问题暴露
4. 在刚才的按钮点击stop

![start_trace](http://ofaeieqjq.bkt.clouddn.com/start_trace.png)

## 分析

卡顿根本原因由于Ui渲染被阻塞造成，它运行在主线程， 因此主线程也被称为UI线程。
原因可能有很多，常见的有主线程中执行速度较慢的I/O操作造成CPU等待， 或主线程有其他大量任务占满了CPU， 比如程序没写好死循环了 - - 。

---

那么我们在这边主线程没有做I/O操作， 因此先按 Excl Cpu Time % 排序， 查看CPU占用百分比 。

定位到 no.18 的方法占用了30.4%， 点选他之后 ， 看到 【1】main 主线程中后面大部分都变色， 表明是这个method的执行区间。 卡顿罪魁祸首就是它了！

![cpu_consume_define](http://ofaeieqjq.bkt.clouddn.com/cpu_consume_define.png)


点开方法名左边小箭头， parent表示是被哪个父方法调用， children是表示调用的所有子方法。

点开children分析后面几个参数

1. Incl Cpu Time 表示各子方法的总运行时长, 主要就看他定位具体方法
2. Incl Cpu Time % 就是各子方法的分配到的CPU占比
3. calls+RecurCalls/Total 表示被调用的次数 (大致符合逻辑就行 主要是看不是异常大 判断死循环)
4. 找到性能表现有问题的方法后一个一个双击看下去就可以了, 同时结合上半部分界面的运行时图表也能直观看出资源占用的区间

![child_cunsume_define](http://ofaeieqjq.bkt.clouddn.com/child_cunsume_define.png)

## 优化

看一下排前5的几个方法, 2个属于反射获取Annotation和Filed, 反射是众所周知的慢, 先放过他们, 另外3个我们看一下方法内容

```java

/**
     * 将我们的数据转化为树的节点
     *
     * @param datas
     *
     * @return
     *
     * @throws NoSuchFieldException
     * @throws IllegalAccessException
     * @throws IllegalArgumentException
     */
    private static <T> List<Node> convetData2Node(List<T> datas)
            throws IllegalArgumentException, IllegalAccessException

    {
        List<Node> nodes = new ArrayList<Node>();
        Node node = null;

        for (T t : datas) {
            int id = -1;
            int pId = -1;
            int pnode = -1;
            String label = null;
            Class<? extends Object> clazz = t.getClass();
            Field[] declaredFields = clazz.getDeclaredFields();
            for (Field f : declaredFields) {
                if (f.getAnnotation(TreeNode.class) != null) {
                    f.setAccessible(true);
                    pnode = f.getInt(t);
                }

                if (f.getAnnotation(TreeNodeId.class) != null) {
                    f.setAccessible(true);
                    id = f.getInt(t);
                }
                if (f.getAnnotation(TreeNodePid.class) != null) {
                    f.setAccessible(true);
                    pId = f.getInt(t);
                }
                if (f.getAnnotation(TreeNodeLabel.class) != null) {
                    f.setAccessible(true);
                    label = (String) f.get(t);
                }
                if (pnode != -1 && id != -1 && pId != -1 && label != null) {
                    break;
                }
            }
            node = new Node(pnode, id, pId, label);
            nodes.add(node);
        }

        /*
         * 设置Node间，父子关系;让每两个节点都比较一次，即可设置其中的关系
         */
        for (int i = 0; i < nodes.size(); i++) {
            Node n = nodes.get(i);
            for (int j = i + 1; j < nodes.size(); j++) {
                Node m = nodes.get(j);
                if (m.getpId() == n.getId()) {
                    n.getChildren().add(m);
                    m.setParent(n);
                } else if (m.getId() == n.getpId()) {
                    m.getChildren().add(n);
                    n.setParent(m);
                }
            }
        }

        // 设置图片
        for (Node n : nodes) {
            setNodeIcon(n);
        }
        return nodes;
    }

```



反射创建节点后通过for循环设置节点的上下级关系， 就是这个for循环有问题了。

1. getId和getPid方法
鸿洋大神的Node类是个标准javaBean， 也就是属性全部private通过get, set操作。but， 这样会比直接读属性慢10倍左右， 其实也没啥大不了， but 如果被调用很多次。。。我把 id 和 pid 改成public 并删掉 get set方法

2. ArrayList.size()
这个也很简单，直接在for循环里定义下变量。改完后的for循环如下

```java

        /*
         * 设置Node间，父子关系;让每两个节点都比较一次，即可设置其中的关系
         */
        for (int i = 0, a = nodes.size(); i < a; i++) {
            Node n = nodes.get(i);
            for (int j = i + 1; j < a; j++) {
                Node m = nodes.get(j);
                if (m.pId == n.id) {
                    n.getChildren().add(m);
                    m.setParent(n);
                } else if (m.id == n.pId) {
                    m.getChildren().add(n);
                    n.setParent(m);
                }
            }
        }

```

效果如下图， 虽然cpu%提高了，但那是由于其他耗时方法不见了，因此总占比就高了，具体看children分类下面，那3个耗时较长的方法已经不见了，self分类的时间也有所降低。 因此incl Cpu Time从2938降低到了1096毫秒，效果还是比较显著的

![after_optimize](http://ofaeieqjq.bkt.clouddn.com/after_optimize.png)


虽然方法优化完了，但还是有1秒的卡顿存在， 因此我后来又改了下把转换节点的操作放到子线程执行了，这就不详细讲了。

## 总结

1. 使用Android Device Monitor工具， 捕获问题发生时的方法执行信息
2. 按Excl Cpu Time % 排序， 找到需要优化的方法
3. 查看方法的children列表， 分析Incl Cpu Time 和 calls+RecurCalls/Total是否合理
4. 找到并修改方法
5. 再次捕获信息， 对比分析
