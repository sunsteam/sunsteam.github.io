---
layout:     post
title:      "Transition 总结"
subtitle:   ""
date:       2017-05-10 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - Material Design
---

参考: [开始使用 Transitions（过渡动画） (part 1)](http://blog.csdn.net/k_tiiime/article/details/44901161)

使用Material风格的动画，在SDK 21+ 实现更好效果

虽然在上个版本中已经引入Activity 和 Fragment 动画(通过 Activity#overridePendingTransition() 和 FragmentTransaction#setCustomAnimation() 方法)，但是动画的对象只能是Activity/Fragment整体。而新的 API 将这个特性延伸，可以为每个 View 单独设置动画，甚至可以在两个独立的 Activity/Fragment 容器内共享某些 View。

## 准备工作

在`values-21`文件夹下建立`style.xml`并加入下列属性或直接继承Material的Theme

```xml
<!-- 启用transitions框架和管理器，TransitionManger会默认在界面切换时使用CrossFade效果，该属性包含ActivityTransition  -->
<item name="android:windowContentTransitions">true</item>
<!-- 启用Transition框架,但没有默认的动画, 需要通过bundle设定动画，设定false则从ContentTransition中关闭此效果-->
<item name="android:windowActivityTransitions">true</item>
<!--是否允许动画重叠-->
<item name="android:windowAllowEnterTransitionOverlap">false</item>
<item name="android:windowAllowReturnTransitionOverlap">false</item>
```

如果Theme直接继承自Material主题,那么`windowActivityTransitions`会默认开启而`windowContentTransitions`会默认关闭，原因是自动的淡入淡出动画可能会对某些Application造成困扰，因此它是默认关着的，但允许直接使用`ActivityOptions.makeSceneTransitionAnimation`通过bundle使用Transition。

## Content Transition

核心方法

```java
ActivityOptionsCompat transitionActivityOptions = ActivityOptionsCompat.makeSceneTransitionAnimation(this, pairs);
startActivity(i, transitionActivityOptions.toBundle());
```

Activity内使用`getWindow().setEnterTransition(transition);`定义动画的执行期间。对应的退出和回到界面等各种期间。**下面这些方法必须在setContentView方法后才能设置，否则Transition框架无法找到界面视图**

- setExitTransition() - 当 A 启动 B 时， A 中 views 离开场景的退出过渡动画

- setEnterTransition() - 当 A 启动 B 时， B 中 views 进入场景的进入过渡动画

- setReturnTransition() - 当 B 返回 A 时， B 中 views 离开场景的返回过渡动画

- setReenterTransition() - 当 B 返回 A 时， A 中 views 进入场景的重入过渡动画

执行动画时就找到这些`View`,然后根据定义的动画类型（包括`fade`,`slide`,`explode`三种）生成动画。 动画类型可以通过代码New一个或者使用TransitionInflater创建。 最后要用`supportFinishAfterTransition();`代替`finish`。

```java
private void setupWindowAnimations() {
    Transition transition;

    if (type == TYPE_PROGRAMMATICALLY) {
        transition = buildEnterTransition();
    }  else {
        transition = TransitionInflater.from(this).inflateTransition(R.transition.explode);
    }
    getWindow().setEnterTransition(transition);
}

private Transition buildEnterTransition() {
    Explode enterTransition = new Explode();
    enterTransition.setDuration(getResources().getInteger(R.integer.anim_duration_long));
    return enterTransition;
}

private void setupLayout() {
    findViewById(R.id.exit_button).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            supportFinishAfterTransition();
        }
    });
}
```

让我们来分析以下具体发生了什么：

- 首先ActivityA启动了ActivityB；
- Transition Framework找到A的退出动画（Slide）并且应用；
- Transition Framework找到B的进入动画（Explode）并且应用；
- 返回事件被触发后，Transition Framework执行进入动画和退出动画的逆向过程（但是如果我们定义了returnTransition和reenterTransition动画，返回效果将会按照我们定义的动画执行）。

### Transition动画的共享View

`makeSceneTransitionAnimation`包含1一个可变参数，使用一个`Pair<View, String>[]`容器包裹。如果切换的界面中的有共享的组件, 可以将他们标记出来使动画效果更好。

- `String`代表的是View的TransitionName,是识别共享组件的唯一标识,不同界面共享组件的TransitionName必须完全相同。也可以在XML中通过`android:transitionName`定义

```java
/**
     * Sets the name of the View to be used to identify Views in Transitions.
     * Names should be unique in the View hierarchy.
     *
     * @param transitionName The name of the View to uniquely identify it for Transitions.
     */
    public final void setTransitionName(String transitionName) {
        mTransitionName = transitionName;
    }

    /**
     * Returns the name of the View to be used to identify Views in Transitions.
     * Names should be unique in the View hierarchy.
     *
     * <p>This returns null if the View has not been given a name.</p>
     *
     * @return The name used of the View to be used to identify Views in Transitions or null
     * if no name has been given.
     */
    @ViewDebug.ExportedProperty
    public String getTransitionName() {
        return mTransitionName;
    }
```

下面的代码将status bar和底部的navigation bar加入共享组件，然后加上用户自定义的共享组件后返回。

```java
/**
 * Helper class for creating content transitions used with {@link android.app.ActivityOptions}.
 */
class TransitionHelper {

    /**
     * Create the transition participants required during a activity transition while
     * avoiding glitches with the system UI.
     *
     * @param activity The activity used as start for the transition.
     * @param includeStatusBar 如果是false,state bar不会执行动画。If false, the status bar will not be added as the transition
     *        participant.
     * @return All transition participants.
     */
    public static Pair<View, String>[] createSafeTransitionParticipants(@NonNull Activity activity,boolean includeStatusBar, @Nullable Pair... otherParticipants) {
        // Avoid system UI glitches as described here:
        // https://plus.google.com/+AlexLockwood/posts/RPtwZ5nNebb
        View decor = activity.getWindow().getDecorView();
        View statusBar = null;
        if (includeStatusBar) {
            statusBar = decor.findViewById(android.R.id.statusBarBackground);
        }
        View navBar = decor.findViewById(android.R.id.navigationBarBackground);

        // Create pair of transition participants.
        List<Pair> participants = new ArrayList<>(3);
        addNonNullViewToTransitionParticipants(statusBar, participants);
        addNonNullViewToTransitionParticipants(navBar, participants);
        // only add transition participants if there's at least one none-null element
        if (otherParticipants != null && !(otherParticipants.length == 1
                && otherParticipants[0] == null)) {
            participants.addAll(Arrays.asList(otherParticipants));
        }
        return participants.toArray(new Pair[participants.size()]);
    }

    private static void addNonNullViewToTransitionParticipants(View view, List<Pair> participants) {
        if (view == null) {
            return;
        }
        participants.add(new Pair<>(view, view.getTransitionName()));
    }

}
```

### 动画执行原理浅析

详细: [深入理解Content Transition (part 2) ](http://blog.csdn.net/k_tiiime/article/details/45535109)

Activity A调用startActivity() 启动Activity B 时的Transition事件流程如下:

1. Activity A 调用 startActivity()

    - A 的退出 transition 捕获 A 中 transitioning views 的起始状态。
    - 框架将 A 中所有 transitioning views 设置为不可见。
    - 在下一个画面中，A 的退出 transition 捕获 A 中所有 transitioning views 的结束状态。
    - A 的退出 transition 比较每一个 transitioning view 开始和结束状态的不同，并基于这些信息创建一个 Animator，最后运行 Animator 将所有 transitioning views移出场景。

2. Activity B 被启动
    - 框架遍历 B 的 view 层次结构， 确定当 B 的进入 transition 运行后有哪些transitioning views 会进入场景。
    - B 的进入 transition 捕获 B 中 transitioning views 的起始状态。
    - 框架将 B 中所有 transitioning views 设置为可见
    - 在下一个画面中，B 的进入 transition 捕获 B 中所有 transitioning views 的结束状态。
    - B 的进入 transition 比较每一个 transitioning view 开始和结束状态的不同，并基于这些信息创建一个 Animator，最后运行 Animator 将所有 transitioning views移入场景。

**框架通过在可见和不可见之间切换每个 transitioning view 的可见性来保证content transition 能够获得用来构建目标动画所需要的状态信息。** 显然所有的 content Transition 对象至少要能够获取和记录每个 transitioning view起始和结束状态的可见性。还好 Visibility 这个抽象类已经提供了这个功能 :需要创建并返回 (让 view 进入/退出场景的) Animator 的 Visibility子类只需要实现 onAppear() 和 onDisappear() 这两个工厂方法。

从 API 21 开始，有三个已经写好的 Visibility 实现( Fade, Slide 和 Explode)，可以使用它们来构建 Activity 和 Fragment 的 content transitions。有必要的话也可以自己实现 Visibility 类达到想实现的效果。

这个递归调用很简单: 框架遍历树的每一层，直到找到一个可见的 leaf view (子视图)或者一个transition group。Transition groups 本质上允许我们在 Activity/Fragment 的 transition期间将全部 ViewGroups 当作一个整体执行过渡动画。如果一个 ViewGroup 的isTransitionGroup () 方法返回值为 true，它和它的子视图会被当作一个整体来执行过渡动画。否则，这个递归搜索会继续执行下去， 这个 ViewGroup 的子视图在动画期间会执行自己的独立的过渡动画。搜索最终会返回一个全部由 content transition执行动画的 transitioning views 集合 。

如果 ViewGroup 有一个非空的 background drawable 或者非空的默认 transition name 那么isTransitionGroup() 将返回 true

当 transition 运行时，任何在 content Transition 对象中被明确地 added 或excluded 的 view 也会被考虑。

#### Transition Group 的实际应用

[Video 2.1](http://www.androiddesignpatterns.com/assets/videos/posts/2014/12/15/games-opt.mp4) 展示了 transition groups 的效果。在 enter transition ，用户头像是作为一个单独的 View 进入屏幕，return transition 时却是和包含它的 parentViewGroup 一起消失。在 Google Play Games 里可能用了一个 transition group来实现在返回前一个 activity 时，让当前场景拦腰斩断的效果。

有时 transition groups 还被用来修复 Activity/Fragment transitions 中诡异的 bugs。例如，[Video 2.2](http://www.androiddesignpatterns.com/assets/videos/posts/2014/12/15/webview-opt.mp4)中: 调用 Activity 显示了一个电台司令专辑图的网格布局，被调用 Activity 展示了一个背景标题图，一个共享元素专辑封面图还有一个 WebView。这个 App 使用了一个和 Google Play Games app 类似的 return transition，将背景图和底部的WebView 分别推出屏幕。然而这里有个小故障导致 WebView 不能流畅的退出屏幕。

好吧，错误在哪呢？原来 WebView 是一个 ViewGroup，因此在默认情况下 WebView 不会被当作 transitioning view的。当 return transition 被执行时，WebView 会被完全忽略，直到过渡动画结束才会被移除屏幕。找到问题就好解决了，只要在 return transition 开始前调用 webView.setTransitionGroup(true)就能修复这个bug。


### Content Transition 总结

- Content transition 决定了 Activity/Fragment 中非共享元素视图(被称为 transitioning views)在 Activity/Fragment transition 期间如何进入或退出场景。
- Content transitions 被触发是因为它的 transitioning views 可见性改变 ，并且应该总是继承Visibility 这个抽象类。
- Transition groups 可以让我们在 content transition 期间将 ViewGroups当作一个整体执行过渡动画。

## Share Elements Transition

参考：[深入理解 Shared Element Transition (part 3a) ](http://blog.csdn.net/k_tiiime/article/details/45535093)

提供一个类似View在2个界面之间传递的动画，当然他并没有在页面间移动，而是被调用的界面B在视觉上模拟了传递的过程。当 A 启动 B ，框架收集 A中共享元素的所有相关信息，并传递给 B。接下来 B 使用这些信息初始化共享元素视图的起始状态(它们在 A 中时对应的大小，位置和外观)。Transition 开始时，B 。Transition 的执行过程中，框架将 B 的 Activity 窗口逐渐显示，直到 B中共享元素结束动画窗口变为不透明。

可以通过下面的Window/Fragment 方法设置：

- `setSharedElementEnterTransition()` - B 的 进入 共享元素 transition ，执行将共享元素视图 从 A 中的起始位置移动到它在 B 中的最终位置的动画。
- `setSharedElementReturnTransition()` - B 的 返回 共享元素 transition ，执行将共享元素视图 从 B 中起始位置移动到它在 A 中的最终位置的动画。

Activity Transition API 也提供了setSharedElementExitTransition() 和 setSharedElementReenterTransition()这两个方法来设置 退出/重入 共享元素过渡，虽然通常来说是不必要的。**并且如果只有退出动画而不设定进入动画，Transition框架会Crash。**

[Video 3.1](http://www.androiddesignpatterns.com/assets/videos/posts/2015/01/12/music-opt.mp4) 展示了在 Google Play Music 中是怎样使用共享元素 transition的。这个 transition 包含两个元素：一个 ImageView和它的父视图 CardView。Transition 期间，CardView 会扩展到全屏或收缩回原状，ImageView 能在这两个 Activity 里无缝的衔接。


### 动画执行原理浅析

和 Content Transitions 的底层相似，框架通过在运行时明确的更改每个 共享元素视图 的属性将这个状态信息提供给 共享元素 Transition 。更准确地说 Activity A 启动 Activity B 时将会出现以下事件(Activities 和 Fragments 的 退出/返回/重入 transition 过程中出现事件序列相似)：

- Activity A 调用 startActivity() 构造，测量，布局了一个最初背景色为透明的半透明窗口 Activity B 。
- 框架将 B 中每一个共享元素视图复位到对应的原来在 A 中时的位置，接着 B 的进入 transition 捕获 B 中所有共享元素视图的起始状态。
- 框架将 B 中每一个共享元素视图复位到对应的在 B 的最终位置，接着 B 的进入 transition 捕获 B 中所有共享元素视图的结束状态。
- B 的进入 transition 比较所有共享元素视图的起始和结束状态，根据它们的不同创建一个 Animator。
- 框架命令 A 隐藏共享元素视图，并运行返回的 Animator。B 中的共享元素视图到位之后，B 的窗口背景在 A 上逐渐显示，直到 B完全的显示出来，transition 运行完毕。

**共享元素 transition是根据每个共享元素视图的位置，大小和外观的变化来调节的**。从 API 21 开始，框架提供了几个自定义共享元素场景切换动画的 Transition 实现。

- *ChangeBounds* - 捕获共享元素布局边界根据不同构造动画。Activity/Fragment 间会有大小 或/和 位置不同。
- *ChangeTransform* - 捕获共享元素缩放和角度，根据不同构建动画。ChangeTransform 还有一个超赞的特性，它可以检测并处理共享元素父视图过渡期间的改变。当共享元素父视图有一个不透明背景，在场景变换过程中默认被选为transitioning view 时，ChangeTransform 就有了用武之地。如果它检测出共享元素父视图已被 content transition 更改，就会将共享元素提取出来，单独执行共享元素的动画。[StackOverflow answer](http://stackoverflow.com/questions/26899779/enter-transition-on-a-fragment-with-a-shared-element-targets-the-shared-element) 这里有 George Mount的详细说明。
- *ChangeClipBounds* - 捕获共享元素的 clip bounds(剪辑边界) ，根据不同构建动画。
- *ChangeImageTransform* - 捕获共享元素 ImageView 的变换矩阵( transform matrices) ，根据不同构建动画。结合 ChangeBounds，可以让 ImageView无缝的改变大小，形状和ImageView.ScaleType 。
- *@android:transition/move* - 一个 TransitionSet ，同时执行上面四种transition 。如果没有明确的声明 进入/返回 共享元素transition ，框架会默认运行这个 transition。


### ViewOverlay

如果想要完全理解共享元素 transition 的运作，我们必须先说说共享元素 overlay。可能不是很明显，**共享元素默认是在整个窗口视图层的顶层 ViewOverlay上绘制**。简单介绍下 ，ViewOverlay 这个类是在 API 18 中为了方便在视图层顶层绘制引入的。添加到视图 ViewOverlay 之中的Drawable和 view (甚至是一个 ViewGroup 的子类) ，将会被绘制在视图的最上层。这就解释了框架为什么默认选择在窗视图层的 ViewOverlay 中绘制共享元素。共享元素视图应该是贯穿整个 transition 的焦点；如果 transitioning views 意外的绘制在共享元素之上就会破坏这个效果。注意，将共享元素绘制在整个图层最顶层也有一些负面效果。有可能会将共享元素绘制在 System UI 之上(比如 status bar, navigation bar还有 action bar)。

虽然共享元素默认绘制在共享元素的 ViewOverlay 之中，但是框架也提供了关闭 overlay 的方法，只要调用`Window#setSharedElementsUseOverlay(false)`就可以了。如果你关闭了 overlay，要留意这样做可能会引起的副作用。

例如，[Video 3.2](http://www.androiddesignpatterns.com/assets/videos/posts/2015/01/12/overlay-opt.mp4)执行了一个简单的共享元素 transition 两次，一次开启和一次关闭 共享元素 overlay 。第一次达到了预期想要的结果，第二次关闭 overlay 后运行的效果不理想。Transition view 从底部向上移入调用 Activity 的 content view 时挡住了部分 共享元素 ImageView 。虽然可以改变在 View 上绘制视图的顺序或者通过在共享元素 parent 里调用setClipChildren(false) 这些旁门左道来修复问题，但是与可能带来的维护问题相比真是得不偿失。总之，除非你感觉必须要关掉共享元素 overlay 才能达到你想要的效果，其他情况尽量不要关闭它，这样会保持代码简洁，并且共享元素 transition 效果更引人注目。

注意，和Activity Transition 不同，Fragment Transition期间共享元素默认不在 ViewOverlay 中绘制。尽管如此，你仍可以使用 ChangeTransformtransition 来达到相似的效果，如果它检测到父视图改变了，就会把共享元素绘制在ViewOverlay 的顶层。相应的，需要关闭这个功能则设置`android:reparentWithOverlay="false"`。


## 动画使用中的问题

Transitions 必须捕获目标 View 的起始和结束状态来构建合适的动画。因此，如果框架在共享元素获得它在调用它的 App 所给定的大小和位置前启动共享元素的过渡动画，这个 Transition 将不能正确捕获到共享元素的结束状态值,生成动画也会失败。Transition 开始前，能否计算出正确的共享元素的结束值主要依靠两个因素:(1) 调用共享元素 Activity布局的复杂度和层次结构的深度 (2)调用共享元素Activity载入数据消耗的时间。布局越复杂，在屏幕上确定共享元素的大小位置耗时越长。同样，如果调用共享元素的 Activity 依赖一个异步的数据载入，框架仍有可能会在数据载入完成前自动开始共享元素 Transition。

- 共享元素存在于 Activity 托管的 Fragment 中 ( a Fragment hosted by the calledactivity)。FragmentTransactions 在commit后不会被立即执行;它们被安排到主线程等待执行。因此，如果共享元素存在的 Fragment 的视图层和FragmentTransaction没有被及时执行，框架有可能在共享元素被正确测量大小和布局到屏幕前启动共享元素 Transition。当然，许多应用通过调用`FragmentManager#executePendingTransactions()`来避开这个问题，这样会强制立即执行FragmentTransactions而不是异步。

- 共享元素是一个高分辨率的图片。给 ImageView 设置一个超过其初始化边界的高分辨率图片，最终可能会导致在这个视图层里额外的布局传递，由此增加在共享元素准备好前就启动 Transition 的几率。流行的异步图片处理库比如 Volley 和 Picasso ，也不能可靠的解决这个问题:框架不能预先了解图片是要被下载，缩放还是在后台线程中从磁盘读取，只是不管图片是否处理完毕就启动共享元素 Transition。

- 共享元素依赖于异步的数据加载如果共享元素所需的数据是通过AsyncTask，AsyncQueryHandler,Loader或者其他类似的东西加载，在它们最终返回数据前 Activity就能被确定，框架仍有可能在数据返回主线程前启动 Transition

### postponeEnterTransition() and startPostponedEnterTransition()

现在你可能会想：如果有办法能让暂时延迟 Transition 的使用，直到我们确定了共享元素的确切大小和位置才使用它就好了。幸好Activity Transitions API为我们提供了解决方案。

在 Activity 的onCreate()中调用`postponeEnterTransition()`方法来暂时阻止启动共享元素 Transition。之后，你需要在共享元素准备好后调用`startPostponedEnterTransition()`来恢复过渡效果。常见的模式是在一个OnPreDrawListener中启动延时 Transition，它会在共享元素测量和布局完毕后被调用。

**注意**,postponeEnterTransition()和startPostponedEnterTransition()只对 Activity Transition起作用。在Fragment中需要通过add并在合适的时候show来解决这个问题，详细信息可以在这里找到[在Fragment中延迟调用Transition](http://stackoverflow.com/questions/26977303/how-to-postpone-a-fragments-enter-transition-in-android-lollipop)。

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // Postpone the shared element enter transition.
    postponeEnterTransition();

    // TODO: Call the "scheduleStartPostponedTransition()" method
    // below when you know for certain that the shared element is
    // ready for the transition to begin.
}

/**
 * Schedules the shared element transition to be started immediately
 * after the shared element has been measured and laid out within the
 * activity's view hierarchy. Some common places where it might make
 * sense to call this method are:
 *
 * (1) Inside a Fragment's onCreateView() method (if the shared element
 *     lives inside a Fragment hosted by the called Activity).
 *
 * (2) Inside a Picasso Callback object (if you need to wait for Picasso to
 *     asynchronously load/scale a bitmap before the transition can begin).
 *
 * (3) Inside a LoaderCallback's onLoadFinished() method (if the shared
 *     element depends on data queried by a Loader).
 */
private void scheduleStartPostponedTransition(final View sharedElement) {
    sharedElement.getViewTreeObserver().addOnPreDrawListener(
        new ViewTreeObserver.OnPreDrawListener() {
            @Override
            public boolean onPreDraw() {
                sharedElement.getViewTreeObserver().removeOnPreDrawListener(this);
                startPostponedEnterTransition();
                return true;
            }
        });
}
```

这里还有第二种方法可以延迟共享元素 **返回** Transition，在调用Activity的onActivityReenter() 方法中延缓返回 Transition。

```java

/**
 * Don't forget to call setResult(Activity.RESULT_OK) in the returning
 * activity or else this method won't be called!
 */
@Override
public void onActivityReenter(int resultCode, Intent data) {
    super.onActivityReenter(resultCode, data);

    // Postpone the shared element return transition.
    postponeEnterTransition();

    // TODO: Call the "scheduleStartPostponedTransition()" method
    // above when you know for certain that the shared element is
    // ready for the transition to begin.
}
```

尽管添加延时可以让共享元素 Transition 更加流畅准确，但是你也要知道在应用中引入共享元素 Transition 的延迟还有一些副作用：

- 调用postponeEnterTransition后不要忘记调用startPostponedEnterTransition。忘记调用startPostponedEnterTransition会让你的应用处于死锁状态，用户无法进入下个Activity。

- 不要将共享元素 Transition 延迟设置到1s以上。延迟时间过长会在应用中产生不必要的卡顿，影响用户体验。
