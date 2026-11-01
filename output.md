::: {.header-element}
::: {.read-progress}
:::
:::

::: {.header-element}
::: {.header-sidebar-menu}
::: {style="padding-left: 1px;"}

:::
:::
:::

::: {.header-actions}
[ []{.iconfont-archer .search-icon aria-hidden="true"}
]{.header-search-btn .header-element .popup-trigger role="button"
aria-label="Search" tooltip="Search"} [ [◐]{aria-hidden="true"}
]{.header-theme-btn .header-element role="button" aria-label="Theme"
tooltip="Theme"} [ [EN](javascript:void(0)){.lang-text}
]{.header-lang-btn .header-element tooltip="Language"}
:::

::: {.banner}
::: {.blog-title .header-element}
[Android Performance](/)
:::

::: {.post-title .header-element}
[Android Perfetto 系列 8：深入理解 Vsync 机制与性能分析](#){.post-name}
:::
:::

::: {.donate-popup .donate-popup--hidden}
::: {.donate-popup__title}
Buy Me A Coffee
:::

::: {.donate-popup__content}
::: {.donate-popup__content-description}
如果觉得文章还不错，打赏一杯咖啡吧.
:::

![Wechat](/assets/donate-wechat.webp "Wechat"){.donate-popup__content-qrCode}
![Alipay](/assets/donate-alipay.png "Alipay"){.donate-popup__content-qrCode}
:::
:::

::: {.footer-fixed-btn .footer-fixed-btn--hidden .donate-btn title="Donate to the author"}
[¥]{aria-hidden="true"}
:::

::: {.footer-fixed-btn .footer-fixed-btn--hidden .back-top}
<div>



</div>
:::

::: {.wrapper}
::: {.site-intro style="    height:50vh;
"}
::: {.site-intro-placeholder}
:::

![](/intro/post-bg.jpg){.site-intro-img sizes="100vw"}

::: {.site-intro-meta}
Android Perfetto 系列 8：深入理解 Vsync 机制与性能分析 {#android-perfetto-系列-8深入理解-vsync-机制与性能分析 .intro-title}
======================================================

::: {.post-intros}
::: {.post-intro-tags}
[ []{.iconfont-archer .post-category-icon aria-hidden="true"} [ Android
]{.post-category-text} ]{.post-category categories="Android"}
[Android](javascript:void(0);){.post-tag}
[Perfetto](javascript:void(0);){.post-tag}
[Systrace](javascript:void(0);){.post-tag}
[Performance](javascript:void(0);){.post-tag}
[RenderThread](javascript:void(0);){.post-tag}
:::

::: {.post-intro-read}
Word count: [5.8k]{.post-count .word-count}Reading time: [23
min]{.post-count .reading-time}
:::

::: {.post-intro-meta}
[]{.iconfont-archer .post-intro-calander}
[2025/08/05]{.post-intro-time} [ []{.iconfont-archer .share-icon}
[Share]{.share-text} ]{.share-wrapper}

-   
    ::: {.share-qrcode}
    :::
-   
-   
-   
-   
:::
:::
:::
:::

![loading](/assets/loading.svg){.loading}

::: {.container .container-unloaded}
::: {.main .post-page role="main"}
在 Perfetto 里看到 `vsync-app`、`vsync-sf` 和 App 的
`doFrame`，很容易把三条轨道当成同一个时刻。这篇沿一次帧请求看 VSync
怎样从 SurfaceFlinger 的调度进入 App，接着怎样和
RenderThread、Buffer、FrameTimeline 对齐。

示例以 120 Hz 设备为主。相位配置、是否启用硬件 VSync，以及 App
是否请求下一帧都会改变轨道形态；本文先分清每个信号代表什么，再谈延迟。

> **注：本文以 Android 17（API 37）为当前技术参照，涉及 Android 13
> 的地方用于说明关键机制的引入；文中代码以 AOSP main
> 的"签名对齐精简摘录"为主，少量位置使用 `...` 省略非主线逻辑，不能当成
> Android 17 发布分支的逐行源码，请以目标分支为准。**

[]{#more}

[](#本文目录 "本文目录"){.headerlink}本文目录
---------------------------------------------

-   [系列文章目录](#%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E7%9B%AE%E5%BD%95)
-   [什么是 Vsync](#%E4%BB%80%E4%B9%88%E6%98%AF-Vsync)
-   [Android 中 Vsync
    的基本工作原理](#Android-%E4%B8%AD-Vsync-%E7%9A%84%E5%9F%BA%E6%9C%AC%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86)
-   [在 Perfetto 中观察
    Vsync](#%E5%9C%A8-Perfetto-%E4%B8%AD%E8%A7%82%E5%AF%9F-Vsync)
-   [Android App 每一帧是如何基于 Vsync
    工作的](#Android-App-%E6%AF%8F%E4%B8%80%E5%B8%A7%E6%98%AF%E5%A6%82%E4%BD%95%E5%9F%BA%E4%BA%8E-Vsync-%E5%B7%A5%E4%BD%9C%E7%9A%84)
-   [参考文档](#%E5%8F%82%E8%80%83%E6%96%87%E6%A1%A3)
-   [关于我 &&
    博客](#%E5%85%B3%E4%BA%8E%E6%88%91-amp-amp-%E5%8D%9A%E5%AE%A2)

[]{#series}

[](#系列文章目录 "系列文章目录"){.headerlink}系列文章目录
=========================================================

1.  [Android Perfetto
    系列目录](https://www.androidperformance.com/2024/03/27/Android-Perfetto-101/#/Perfetto-%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95)
2.  [Android Perfetto 系列 1：Perfetto
    工具简介](https://www.androidperformance.com/2024/05/21/Android-Perfetto-01-What-is-perfetto/)
3.  [Android Perfetto 系列 2：Perfetto Trace
    抓取](https://www.androidperformance.com/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/)
4.  [Android Perfetto 系列 3：熟悉 Perfetto
    View](https://www.androidperformance.com/2024/05/21/Android-Perfetto-03-how-to-analysis-perfetto/)
5.  [Android Perfetto 系列 4：使用命令行在本地打开超大
    Trace](https://www.androidperformance.com/2025/02/08/Android-Perfetto-04-Open-Big-Trace-With-Command-Line/)
6.  [Android Perfetto 系列 5：Android App 基于 Choreographer
    的渲染流程](https://www.androidperformance.com/2025/03/26/Android-Perfetto-05-Chorergrapher/)
7.  [Android Perfetto 系列 6：为什么是
    120Hz？高刷新率的优势与挑战](https://www.androidperformance.com/2025/04/26/Android-Perfetto-06-Why-120Hz/)
8.  [Android Perfetto 系列 7：MainThread 和 RenderThread
    解读](https://www.androidperformance.com/2025/08/02/Android-Perfetto-07-MainThread-And-RenderThread/)
9.  [Android Perfetto 系列 8：深入理解 Vsync
    机制与性能分析](https://www.androidperformance.com/2025/08/05/Android-Perfetto-08-Vsync/)
10. [Android Perfetto 系列 9：CPU
    信息解读](https://www.androidperformance.com/2025/11/12/Android-Perfetto-09-CPU/)
11. [Android Perfetto 系列 10：Binder
    调度与锁竞争](https://www.androidperformance.com/2025/11/16/Android-Perfetto-10-Binder/)
12. [Android Perfetto 系列 11：PerfettoSQL、Trace Processor
    与回归检测](https://www.androidperformance.com/2026/05/04/Android-Perfetto-11-PerfettoSQL-TraceProcessor-Automation/)
13. [Android Perfetto 系列 12：Trace
    数据流与丢失排查](https://www.androidperformance.com/2026/05/04/Android-Perfetto-12-Trace-Dataflow-And-Loss/)
14. [Android Perfetto 系列 13：Perfetto SDK、Track Event 与 App 现场
    Trace](https://www.androidperformance.com/2026/05/04/Android-Perfetto-13-Track-Event-SDK/)
15. [Android Perfetto 系列 14：heapprofd 与内存
    Profiling](https://www.androidperformance.com/2026/05/04/Android-Perfetto-14-Native-Heap-Heapprofd/)
16. [Android Perfetto 系列 15：Boot Trace、长时间现场 Trace
    与偶发问题抓取](https://www.androidperformance.com/2026/05/04/Android-Perfetto-15-Boot-And-Long-Running-Field-Tracing/)
17. [Android Perfetto 系列 16：GPU、Power Counters
    与硬件瓶颈分析](https://www.androidperformance.com/2026/05/04/Android-Perfetto-16-GPU-Power-Counters/)
18. [Android Perfetto 系列
    17：专项场景自动化、问题清单与平台体系](https://www.androidperformance.com/2026/05/04/Android-Perfetto-17-Engineering-Domain-And-Platform-Tracing/)
19. [Android Perfetto 系列
    18：响应速度实战，从输入事件到首帧反馈](https://www.androidperformance.com/2026/05/04/Android-Perfetto-18-Input-Response-Latency/)
20. [视频(B站) - Android Perfetto
    基础和案例分享](https://www.bilibili.com/video/BV1oi82efE4D/?vd_source=0c6d2191e785de0a36dc21a9da7e664e)
21. [视频(B站) - Android Perfetto 分享 -
    出图类型分享：AOSP、WebView、Flutter + OEM
    系统优化分享](https://www.bilibili.com/video/BV17A6bBLECu/)

如果大家还没看过 Systrace 系列，下面是传送门：

1.  [Systrace
    系列目录](https://www.androidperformance.com/2019/05/26/Android_Systrace_0/#/%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E7%9B%AE%E5%BD%95)
    ： 系统介绍了 Perfetto 的前身 Systrace 的使用，并通过 Systrace
    来学习和了解 Android 性能优化和 Android 系统运行的基本规则。
2.  [个人博客](https://www.androidperformance.com/) ：个人博客，主要是
    Android 相关的内容，也放了一些生活和工作相关的内容。

欢迎大家在 [关于我](https://www.androidperformance.com/about/)
页面加入微信群或者星球，讨论你的问题、你最想看到的关于 Perfetto
的部分，以及跟各位群友讨论所有 Android 开发相关的内容

[]{#what-is-vsync}

[](#什么是-Vsync "什么是 Vsync"){.headerlink}什么是 Vsync {#什么是-Vsync}
---------------------------------------------------------

VSync（Vertical
Synchronization，垂直同步）给帧生产和显示合成提供共同的时间参考。App、SurfaceFlinger
与显示硬件会按各自的调度相位工作，不能把它们画成同一个瞬间。

在没有 Vsync 机制之前，常见问题是屏幕撕裂（Screen
Tearing）。当显示器读取 framebuffer 的同时，GPU
写入了下一帧，就会在同一次刷新中出现上下两部分不一致的画面。

### [](#Vsync-解决什么问题？ "Vsync 解决什么问题？"){.headerlink}Vsync 解决什么问题？ {#Vsync-解决什么问题？}

Android 用 VSync 安排帧回调和合成时点，再用 BufferQueue 与 fence
协调读写：

1.  **显示节拍**：显示硬件提供刷新周期信息，系统也可以根据硬件反馈预测后续时点。
2.  **App 回调**：有待处理帧时，Choreographer 请求
    VSync，在回调中执行已登记的 Input、Animation、Traversal 等工作；GPU
    工作可继续异步运行。
3.  **合成与同步**：SurfaceFlinger 按自己的调度时点取可用
    Buffer；acquire fence 约束何时能安全读取，present fence
    帮助确认显示完成。

以固定 120 Hz 模式为例，名义显示间隔约 8.33 ms。App
的帧可能在更早的周期开始，也可能错过当前预期呈现时点；是否晚呈现要用
FrameTimeline 和合成侧记录判断，不能只看 `queueBuffer` 的时间。

[]{#principle}

[](#Android-中-Vsync-的基本工作原理 "Android 中 Vsync 的基本工作原理"){.headerlink}Android 中 Vsync 的基本工作原理 {#Android-中-Vsync-的基本工作原理}
------------------------------------------------------------------------------------------------------------------

Android 系统的 Vsync
实现比基本概念复杂得多，需要考虑多个不同的渲染组件，以及它们之间的协调工作。

### [](#Vsync-信号的分层架构 "Vsync 信号的分层架构"){.headerlink}Vsync 信号的分层架构 {#Vsync-信号的分层架构}

在 Android 系统中，并不是只有一个简单的 Vsync
信号。系统维护着多个不同用途的 Vsync 信号：

**硬件 Vsync（HW Vsync）**：\
这是最底层的 Vsync 信号，由显示硬件（HWC，Hardware
Composer）产生。它的频率严格对应显示器的刷新率，比如 60Hz 的显示器会每
16.67ms 产生一次 HW Vsync，120Hz 的显示器会每 8.333ms 产生一次。（硬件
Vsync 回调由 HWC/SurfaceFlinger 管理，详见
`frameworks/native/services/surfaceflinger` 相关实现）

但是，HW Vsync
并不是一直开启的。由于频繁的硬件中断会消耗较多的电量，Android
系统采用了一种智能的策略：只有在需要精确同步的时候才开启 HW
Vsync，大部分时间使用软件预测的方式生成 Vsync 信号。

**Vsync-app（应用 Vsync）**：\
这是专门用于驱动应用层渲染的 Vsync 信号。当应用需要进行 UI
更新时（比如用户触摸、动画运行、界面滚动等），应用会向系统申请接收
Vsync-app 信号。

+-----------------------------------+-----------------------------------+
|     1                             |     // frameworks/base/core/jav   |
|     2                             | a/android/view/Choreographer.java |
|     3                             |     private voi                   |
|     4                             | d scheduleFrameLocked(long now) { |
|     5                             |         if (!mFrameScheduled) {   |
|     6                             |                                   |
|     7                             |           mFrameScheduled = true; |
|     8                             |             if (USE_VSYNC) {      |
|     9                             |                                   |
|     10                            |    // 向系统申请下一个 Vsync 信号 |
|                                   |                 mDisp             |
|                                   | layEventReceiver.scheduleVsync(); |
|                                   |             }                     |
|                                   |         }                         |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

Vsync-app
是按需申请的。如果应用界面是静态的，没有任何动画或用户交互，那么应用不会申请
Vsync-app 信号，系统也就不会为这个应用生成 Vsync 事件。

**Vsync-sf（SurfaceFlinger Vsync）**：\
这是专门用于驱动 SurfaceFlinger 进行图层合成的 Vsync
信号。SurfaceFlinger 是 Android
系统中负责将所有应用的图层合成为最终画面的服务。

**Vsync-appSf（应用-SurfaceFlinger Vsync）**：\
Android 13 引入的新信号类型。为消除旧设计中 sf EventThread 既唤醒
SurfaceFlinger 又服务部分 Choreographer
客户端带来的时序歧义，系统将两类职责分离：`vsync-sf` 专注唤醒
SurfaceFlinger，`vsync-appSf` 面向需要与 SurfaceFlinger 同步的客户端。

[]{#perfetto-observe}

[](#在-Perfetto-中观察-Vsync "在 Perfetto 中观察 Vsync"){.headerlink}在 Perfetto 中观察 Vsync {#在-Perfetto-中观察-Vsync}
---------------------------------------------------------------------------------------------

Perfetto trace 中包含多个与 Vsync 相关的 Track，理解这些 Track
的含义有助于分析性能问题。

**在 SurfaceFlinger 进程中**：

1.  **vsync-app**\
    这份 Trace 中该 counter 在 0 和 1
    之间变化，可用变化时点辅助观察调度节奏。它不是某个指定 App
    已收到回调的证明。\
    ![image-20250811221826847](/images/Android-Perfetto-08-Vsync/image-20250811221826847.webp)

2.  **vsync-sf**\
    显示 SurfaceFlinger Vsync 信号状态。无 Vsync Offset 时与 `vsync-app`
    同步变化。\
    ![image-20250811221902646](/images/Android-Perfetto-08-Vsync/image-20250811221902646.webp)

3.  **vsync-appSf**\
    Android 13+ 新增，服务于需要与 SurfaceFlinger 同步的特殊
    Choreographer 客户端。\
    ![image-20250811222036489](/images/Android-Perfetto-08-Vsync/image-20250811222036489.webp)

4.  **HW\_VSYNC**\
    这条 counter 记录硬件 VSync 的启用状态，值为 1/0
    时分别表示开启/关闭。不要把每次状态变化当成一次 App 帧回调。\
    ![image-20250811222159253](/images/Android-Perfetto-08-Vsync/image-20250811222159253.webp)

**在应用进程中**：

`FrameDisplayEventReceiver.onVsync` Slice Track：\
显示应用接收 Vsync 信号的时间点。该事件连接通过 Binder 建链、通过
`BitTube`/Looper 通道分发事件，时间可能略晚于 SurfaceFlinger 中的
`vsync-app`。

![image-20250918220632473](/images/Android-Perfetto-08-Vsync/image-20250918220632473.webp)

`UI Thread` Slice Track：\
包含 `Choreographer#doFrame` 及相关的 Input、Animation、Traversal 等
Slice。每个 `doFrame` 对应一帧的处理工作。

![image-20250918220709655](/images/Android-Perfetto-08-Vsync/image-20250918220709655.webp)

`RenderThread` Slice Track：\
包含 `DrawFrame`、`syncAndDrawFrame`、`queueBuffer` 等
Slice，对应渲染线程工作。

![image-20250918220730872](/images/Android-Perfetto-08-Vsync/image-20250918220730872.webp)

[]{#app-frames}

[](#Android-App-每一帧是如何基于-Vsync-工作的 "Android App 每一帧是如何基于 Vsync 工作的"){.headerlink}Android App 每一帧是如何基于 Vsync 工作的 {#Android-App-每一帧是如何基于-Vsync-工作的}
------------------------------------------------------------------------------------------------------------------------------------------------

下图把 App 请求回调、主线程执行和 SurfaceFlinger
合成放在一条时间线上。不同帧可能跳过某些阶段，图中箭头表示调查顺序，不表示所有工作都在一个
VSync 周期内完成。

![image-20250918221821265](/images/Android-Perfetto-08-Vsync/image-20250918221821265.webp)

### [](#流程总览（按顺序） "流程总览（按顺序）"){.headerlink}流程总览（按顺序） {#流程总览（按顺序）}

1.  触发重绘/输入：`View.invalidate()`、动画、数据变化或输入事件触发 →
    `ViewRootImpl.scheduleTraversals()` →
    `Choreographer.postCallback(TRAVERSAL)`
2.  申请 Vsync：`Choreographer` 通过
    `DisplayEventReceiver.scheduleVsync()` 申请下一次 Vsync（app 相位）
3.  接收 Vsync：`DisplayEventReceiver.onVsync()` 收到 Vsync
    后，向主线程消息队列投递异步消息
4.  主线程帧处理：`Choreographer.doFrame()`
    按顺序执行五类回调：`INPUT → ANIMATION → INSETS_ANIMATION → TRAVERSAL → COMMIT`
5.  渲染提交：`RenderThread` 执行 `syncAndDrawFrame/DrawFrame`，CPU 记录
    GPU 命令，`queueBuffer` 提交到 BufferQueue
6.  合成显示：有待处理更新时，`SurfaceFlinger` 按 `vsync-sf`
    调度合成（GPU 或 HWC），生成 `present_fence`，输出到显示
7.  帧完成度量：通过 `FrameTimeline`（PresentType/JankType）与
    `acquire/present_fence` 判定是否按期显示

下面分别展开每一步的关键实现与 Perfetto 观测点。

### [](#App-什么时候会申请-Vsync-信号 "App 什么时候会申请 Vsync 信号"){.headerlink}App 什么时候会申请 Vsync 信号 {#App-什么时候会申请-Vsync-信号}

应用并不是时刻都在申请 Vsync 信号的。Vsync
信号是按需申请的，只有在以下情况下，应用才会向系统申请下一个 Vsync：

**常见的触发申请 Vsync 的场景**：

1.  **UI 更新需求**：当 View 调用 `invalidate()` 时
2.  **动画执行**：ValueAnimator、ObjectAnimator 等动画开始时
3.  **用户交互**：触摸事件、按键事件等需要 UI 响应时
4.  **数据变化**：RecyclerView 数据更新、TextView 文本改变等

### [](#App-申请-Vsync-的完整流程 "App 申请 Vsync 的完整流程"){.headerlink}App 申请 Vsync 的完整流程 {#App-申请-Vsync-的完整流程}

当应用需要更新 UI 时，会通过以下流程申请 Vsync 信号：

+-----------------------------------+-----------------------------------+
|     1                             |     // 1. UI 组件请求重绘         |
|     2                             |     // frameworks/base            |
|     3                             | /core/java/android/view/View.java |
|     4                             |     public void invalidate() {    |
|     5                             |                                   |
|     6                             |   // 标记为需要重绘，但不立即执行 |
|     7                             |                                   |
|     8                             |     mPrivateFlags |= PFLAG_DIRTY; |
|     9                             |                                   |
|     10                            |         if (mParent               |
|     11                            | != null && mAttachInfo != null) { |
|     12                            |             // 向父容器请求重绘   |
|     13                            |             mPa                   |
|     14                            | rent.invalidateChild(this, null); |
|     15                            |         }                         |
|     16                            |     }                             |
|     17                            |                                   |
|     18                            |     // 2. ViewRootImpl 调度遍历   |
|     19                            |     // frameworks/base/core/ja    |
|     20                            | va/android/view/ViewRootImpl.java |
|     21                            |     void scheduleTraversals() {   |
|     22                            |                                   |
|     23                            |       if (!mTraversalScheduled) { |
|     24                            |                                   |
|     25                            |       mTraversalScheduled = true; |
|     26                            |             // 关键：向 Choreog   |
|     27                            | rapher 注册回调，等待下一个 Vsync |
|     28                            |                                   |
|     29                            |      mChoreographer.postCallback( |
|     30                            |                                   |
|     31                            |       Choreographer.CALLBACK_TRAV |
|     32                            | ERSAL, mTraversalRunnable, null); |
|     33                            |         }                         |
|     34                            |     }                             |
|     35                            |                                   |
|     36                            |                                   |
|     37                            |    // 3. Choreographer 申请 Vsync |
|     38                            |     // frameworks/base/core/jav   |
|     39                            | a/android/view/Choreographer.java |
|     40                            |     private void postCallback     |
|     41                            | DelayedInternal(int callbackType, |
|     42                            |             Object action,        |
|     43                            | Object token, long delayMillis) { |
|     44                            |         final long                |
|     45                            | now = SystemClock.uptimeMillis(); |
|     46                            |         final                     |
|     47                            | long dueTime = now + delayMillis; |
|     48                            |         mCallb                    |
|     49                            | ackQueues[callbackType].addCallba |
|                                   | ckLocked(dueTime, action, token); |
|                                   |                                   |
|                                   |                                   |
|                                   |       // 重点：不只 TRAVERSAL，任 |
|                                   | 意“到期回调”都可能触发下一帧调度  |
|                                   |         if (dueTime <= now) {     |
|                                   |                                   |
|                                   |         scheduleFrameLocked(now); |
|                                   |         }                         |
|                                   |     }                             |
|                                   |                                   |
|                                   |     private voi                   |
|                                   | d scheduleFrameLocked(long now) { |
|                                   |         if (!mFrameScheduled) {   |
|                                   |                                   |
|                                   |           mFrameScheduled = true; |
|                                   |             if (i                 |
|                                   | sRunningOnLooperThreadLocked()) { |
|                                   |                                   |
|                                   |            scheduleVsyncLocked(); |
|                                   |             } else {              |
|                                   |                                   |
|                                   |       Message msg = mHandler.obta |
|                                   | inMessage(MSG_DO_SCHEDULE_VSYNC); |
|                                   |                                   |
|                                   |        msg.setAsynchronous(true); |
|                                   |                 mHandle           |
|                                   | r.sendMessageAtFrontOfQueue(msg); |
|                                   |             }                     |
|                                   |         }                         |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

> `TRAVERSAL` 仍然是最常见触发源，但从 AOSP main 实现看，并非"只有
> TRAVERSAL 才申请 Vsync"。

### [](#主线程如何监听-Vsync-信号 "主线程如何监听 Vsync 信号"){.headerlink}主线程如何监听 Vsync 信号 {#主线程如何监听-Vsync-信号}

应用主线程通过 `DisplayEventReceiver` 来监听 Vsync
信号。这个过程涉及几个关键步骤：

**1. 建立连接**：

+-----------------------------------+-----------------------------------+
|     1                             |     // frameworks/base/core/jav   |
|     2                             | a/android/view/Choreographer.java |
|     3                             |     private                       |
|     4                             | final class FrameDisplayEventRece |
|     5                             | iver extends DisplayEventReceiver |
|     6                             |             implements Runnable { |
|     7                             |                                   |
|     8                             |         public FrameDispla        |
|     9                             | yEventReceiver(Looper looper, int |
|                                   |  vsyncSource, long layerHandle) { |
|                                   |             s                     |
|                                   | uper(looper, vsyncSource, /* even |
|                                   | tRegistration */ 0, layerHandle); |
|                                   |             // 在构               |
|                                   | 造时建立与 SurfaceFlinger 的连接  |
|                                   |         }                         |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

**2. 接收 Vsync 信号**：

+-----------------------------------+-----------------------------------+
|     1                             |     @Override                     |
|     2                             |     public vo                     |
|     3                             | id onVsync(long timestampNanos, l |
|     4                             | ong physicalDisplayId, int frame, |
|     5                             |                                   |
|     6                             |  VsyncEventData vsyncEventData) { |
|     7                             |         // 接收到 Vsync 信号，    |
|     8                             | 但注意：这里并不直接执行 doFrame  |
|     9                             |                                   |
|     10                            | mTimestampNanos = timestampNanos; |
|     11                            |         mFrame = frame;           |
|     12                            |         mLastVsyncEv              |
|     13                            | entData.copyFrom(vsyncEventData); |
|     14                            |                                   |
|     15                            |         // 关键：将工作           |
|     16                            |  post 到主线程的 MessageQueue 中  |
|     17                            |         Message msg               |
|     18                            | = Message.obtain(mHandler, this); |
|     19                            |                                   |
|                                   | msg.setAsynchronous(true);  // 可 |
|                                   | 越过同步屏障，不保证高于所有消息  |
|                                   |         mHandle                   |
|                                   | r.sendMessageAtTime(msg, timestam |
|                                   | pNanos / TimeUtils.NANOS_PER_MS); |
|                                   |     }                             |
|                                   |                                   |
|                                   |     @Override                     |
|                                   |     public void run() {           |
|                                   |                                   |
|                                   |   // 这里才真正开始执行一帧的工作 |
|                                   |         doFrame(mTimestampNan     |
|                                   | os, mFrame, mLastVsyncEventData); |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

### [](#几个遗留问题 "几个遗留问题"){.headerlink}几个遗留问题

**Q1：为什么不在 `onVsync()` 中直接执行 `doFrame()`？**\
![image-20250918221936675](/images/Android-Perfetto-08-Vsync/image-20250918221936675.webp)

-   线程边界：在 `Choreographer` 场景下，`onVsync()` 回调运行在其绑定的
    Looper（通常就是主线程）；通过消息队列再进入
    `doFrame()`，可统一调度并保持帧处理时序一致
-   调度控制：通过 `sendMessageAtTime()` 精确对齐执行时刻
-   队列语义：进入主线程 MessageQueue，确保与其他高优先级任务协同

**Q2：Vsync 消息来了但主线程在忙，会丢吗？**\
![image-20250918222045731](/images/Android-Perfetto-08-Vsync/image-20250918222045731.webp)

-   不完全是"不会丢"。单次 `scheduleVsync()`
    只请求一次事件；主线程长期繁忙时会出现"跳过多个硬件节拍、最终只处理较新的一帧"的现象。实际分析应结合
    FrameTimeline 判断是否产生可见卡顿。
-   AOSP `DisplayEventDispatcher::processPendingEvents`
    明确会用"后到达的 vsync 覆盖先到达的
    vsync"（只保留最近一次用于分发）。

**Q3：CPU/GPU 是否必须在单个 Vsync 周期内完成？**如果任何一个环节超过 1
个 vsync ，都会导致掉帧？

-   现代 Android 系统采用多缓冲（通常是三缓冲）机制：

-   **应用端**：Front Buffer（显示中）+ Back Buffer（渲染中）+
    可能的第三个 Buffer

-   **SurfaceFlinger 端**：也有类似的缓冲机制

-   这也说明即使应用的某一帧超过了 Vsync 周期，也不一定会立即掉帧。

-   GPU
    可以异步执行；多缓冲允许不同帧的生产与消费重叠，也可能带来额外排队延迟。下图里的
    Buffer counter 只能说明数量变化，不能单独证明「没有掉帧」。

-   判断这次是否出现可见卡顿，还要对齐目标 layer 的 FrameTimeline
    与呈现时间。

![image-20250918222258536](/images/Android-Perfetto-08-Vsync/image-20250918222258536.webp)

**Q4：GPU 和 CPU 是怎么协同的？**：

-   GPU 渲染是异步的，这带来了额外的复杂性：
-   **CPU 工作正常，GPU 成为瓶颈**：即使应用主线程在 Vsync
    周期内完成工作，GPU 渲染耗时过长仍会导致掉帧
-   **GPU Fence 机制**：在 Buffer 被 SF latch 的阶段，关键同步点通常是
    `acquire fence`（Buffer 何时可安全读取）；`present fence`
    记录该帧显示完成的时点。根据系统 `Latch Unsignaled Buffers`
    策略，SurfaceFlinger 在特定条件下可先推进流程，再在实际读取 Buffer
    前等待 fence 信号。

![image-20250918222626100](/images/Android-Perfetto-08-Vsync/image-20250918222626100.webp)

**Q5：VSync 相位会影响什么？**

-   **改变预算位置**：App 与 SurfaceFlinger
    的唤醒时点不同，可能让某些帧赶上更早的合成周期，也可能缩短另一侧可用时间。收益不能统一写成从
    3 个周期缩到 2 个周期。
-   **影响超时边界**：相位调整会改变帧截止时间；App
    本身过慢时，改相位不能替代减少主线程或 GPU 耗时。
-   `VsyncWorkDuration`
    更接近调度预算（workDuration/readyDuration）的可视化，不等价于单一
    appOffset 数值；分析时建议结合 `vsync-app/sf` 与 FrameTimeline
    联动判断。
-   下图中显示的时间段就是我手上的手机配置的 app offset
    （13.3ms）。不同厂商和机型的配置差异很大，而且在 workDuration
    口径下这个值甚至可以超过一个 Vsync
    周期，不要拿别的机器的数值直接对比

![image-20250918222707300](/images/Android-Perfetto-08-Vsync/image-20250918222707300.webp)

### [](#Vsync-Offset-x2F-WorkDuration-的技术实现 "Vsync Offset / WorkDuration 的技术实现"){.headerlink}Vsync Offset / WorkDuration 的技术实现 {#Vsync-Offset-x2F-WorkDuration-的技术实现}

在当前 AOSP main 中，配置入口是 `VsyncConfiguration`
抽象接口，返回的是按场景组织的 `VsyncConfigSet`。实现上 `PhaseOffsets`
属于旧路径，`WorkDuration` 是新路径中更常见的实现之一：

+-----------------------------------+-----------------------------------+
|     1                             |     // framew                     |
|     2                             | orks/native/services/surfacefling |
|     3                             | er/Scheduler/VsyncConfiguration.h |
|     4                             |     class VsyncConfiguration {    |
|     5                             |     public:                       |
|     6                             |         virtual                   |
|     7                             |  ~VsyncConfiguration() = default; |
|     8                             |         virtual VsyncConfigS      |
|     9                             | et getCurrentConfigs() const = 0; |
|     10                            |         v                         |
|     11                            | irtual VsyncConfigSet getConfigsF |
|     12                            | orRefreshRate(Fps fps) const = 0; |
|     13                            |         virtual voi               |
|     14                            | d setRefreshRateFps(Fps fps) = 0; |
|     15                            |         virtual void reset() = 0; |
|                                   |     };                            |
|                                   |                                   |
|                                   |     class WorkDurat               |
|                                   | ion : public VsyncConfiguration { |
|                                   |     public:                       |
|                                   |         explicit Work             |
|                                   | Duration(Fps currentRefreshRate); |
|                                   |                                   |
|                                   |    // 内部根据 sf/app 的 workDura |
|                                   | tion 构造不同场景下的 offset 配置 |
|                                   |     };                            |
+-----------------------------------+-----------------------------------+

**关键概念**：

-   `workDuration/readyDuration`：调度时的"工作预算"和"就绪提前量"，用于计算回调唤醒时刻
-   `app/sf offset`：仍可作为常用分析口径，但本质是配置集合与调度模型共同作用的结果
-   常用口径里"app/sf offset 差值"指两者相位差（通常看
    `|sfOffset - appOffset|`
    的绝对值，具体符号以设备实现与统计口径为准）

### [](#相位差的简化时序示意 "相位差的简化时序示意"){.headerlink}相位差的简化时序示意

下面把 120 Hz 周期画成 8.333 ms，假设 App 能在 3 ms 完成、SurfaceFlinger
再用 2 ms
合成。数字仅用于说明相位可能让一帧赶上不同的呈现窗口，并非设备实测：

**无 Offset（传统方式）**：

-   T0：应用和 SurfaceFlinger 同时接收 Vsync
-   T0+3ms：应用完成渲染
-   T0+8.333ms：下一个 Vsync，SurfaceFlinger 开始合成
-   T0+16.666ms：用户看到画面（总延迟 16.666ms）

**有 Offset（优化方式）**：

-   T0+1ms：应用接收 Vsync-app，开始渲染
-   T0+3ms：应用完成渲染，提交 Buffer
-   T0+4ms：SurfaceFlinger 接收 Vsync-sf，立即开始合成
-   T0+6ms：SurfaceFlinger 完成合成
-   T0+8.333ms：用户看到画面（总延迟 8.333ms）

在这个理想化例子里，改变相位让帧赶上了前一个显示机会。真实设备还要算输入到达、GPU、Buffer
排队和 present fence；不能由这张示意图推出「响应性能提升一倍」。

**实际的时间预算分配**：

以 120Hz 设备为例（8.333ms 周期）：

-   **示意预算**：若 App 的预期完成窗口为 4 ms、合成使用 2
    ms，剩余时间仍受两者调度相位约束，不能简单相加成统一截止线。
-   **跨周期执行**：App、GPU 与 SurfaceFlinger
    可以处理不同帧；多缓冲允许重叠，但不能保证按时呈现。
-   **GPU 限制**：GPU 若持续晚完成，acquire fence
    会延后可安全读取的时间；用实际 GPU 轨道或 fence 数据核对。

**出现晚帧时可继续排查**：

1.  **应用端超时 + Buffer 耗尽**：连续多帧超时导致 BufferQueue 没有可用
    Buffer
2.  **GPU 渲染超时**：即使 CPU 工作正常，GPU 渲染超时也会掉帧
3.  **SurfaceFlinger 超时**：系统级合成超时，影响所有应用
4.  **系统资源竞争**：CPU/GPU/内存等资源被其他进程占用

[](#Vsync-信号的完整代码流程 "Vsync 信号的完整代码流程"){.headerlink}Vsync 信号的完整代码流程 {#Vsync-信号的完整代码流程}
---------------------------------------------------------------------------------------------

Vsync 信号从硬件传递到应用层的完整链路如下。

### [](#按-AOSP-main-分支对齐的关键代码（精简摘录） "按 AOSP main 分支对齐的关键代码（精简摘录）"){.headerlink}按 AOSP main 分支对齐的关键代码（精简摘录） {#按-AOSP-main-分支对齐的关键代码（精简摘录）}

下面片段都按当前 AOSP main
分支的方法签名整理，省略了与主线无关的分支与日志代码。

**1）Choreographer 申请下一次 Vsync（Java）**

+-----------------------------------+-----------------------------------+
|     1                             |     // frameworks/base/core/jav   |
|     2                             | a/android/view/Choreographer.java |
|     3                             |     private voi                   |
|     4                             | d scheduleFrameLocked(long now) { |
|     5                             |         if (!mFrameScheduled) {   |
|     6                             |                                   |
|     7                             |           mFrameScheduled = true; |
|     8                             |             if (USE_VSYNC) {      |
|     9                             |                 if (i             |
|     10                            | sRunningOnLooperThreadLocked()) { |
|     11                            |                                   |
|     12                            |            scheduleVsyncLocked(); |
|     13                            |                 } else {          |
|     14                            |                                   |
|     15                            |       Message msg = mHandler.obta |
|     16                            | inMessage(MSG_DO_SCHEDULE_VSYNC); |
|     17                            |                                   |
|     18                            |        msg.setAsynchronous(true); |
|     19                            |                     mHandle       |
|                                   | r.sendMessageAtFrontOfQueue(msg); |
|                                   |                 }                 |
|                                   |             }                     |
|                                   |         }                         |
|                                   |     }                             |
|                                   |                                   |
|                                   |     pri                           |
|                                   | vate void scheduleVsyncLocked() { |
|                                   |         mDisp                     |
|                                   | layEventReceiver.scheduleVsync(); |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

**2）Choreographer 接收 Vsync（Java）**

+-----------------------------------+-----------------------------------+
|     1                             |     // frameworks/base/core/jav   |
|     2                             | a/android/view/Choreographer.java |
|     3                             |     // TODO(b/11                  |
|     4                             | 6025192): physicalDisplayId is ig |
|     5                             | nored because SF only emits VSYNC |
|     6                             |     // for the interna            |
|     7                             | l display and scheduleVsync only  |
|     8                             | allows requesting internal VSYNC. |
|     9                             |     @Override                     |
|     10                            |     public vo                     |
|     11                            | id onVsync(long timestampNanos, l |
|     12                            | ong physicalDisplayId, int frame, |
|     13                            |                                   |
|     14                            |  VsyncEventData vsyncEventData) { |
|     15                            |                                   |
|     16                            | mTimestampNanos = timestampNanos; |
|     17                            |         mFrame = frame;           |
|     18                            |         mLastVsyncEv              |
|     19                            | entData.copyFrom(vsyncEventData); |
|                                   |         Message msg               |
|                                   | = Message.obtain(mHandler, this); |
|                                   |                                   |
|                                   |        msg.setAsynchronous(true); |
|                                   |         mHandle                   |
|                                   | r.sendMessageAtTime(msg, timestam |
|                                   | pNanos / TimeUtils.NANOS_PER_MS); |
|                                   |     }                             |
|                                   |                                   |
|                                   |     @Override                     |
|                                   |     public void run() {           |
|                                   |                                   |
|                                   |        mHavePendingVsync = false; |
|                                   |         doFrame(mTimestampNan     |
|                                   | os, mFrame, mLastVsyncEventData); |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

**3）JNI 层桥接：`DisplayEventDispatcher`（C++）**

+-----------------------------------+-----------------------------------+
|     1                             |                                   |
|     2                             |  // frameworks/base/core/jni/andr |
|     3                             | oid_view_DisplayEventReceiver.cpp |
|     4                             |                                   |
|     5                             | class NativeDisplayEventReceiver  |
|     6                             | : public DisplayEventDispatcher { |
|     7                             |     public:                       |
|     8                             |         NativeDisplayEventRecei   |
|     9                             | ver(JNIEnv* env, jobject receiver |
|     10                            | Weak, jobject vsyncEventDataWeak, |
|     11                            |                                   |
|     12                            |            const sp<MessageQueue> |
|                                   | & messageQueue, jint vsyncSource, |
|                                   |                 jint event        |
|                                   | Registration, jlong layerHandle); |
|                                   |                                   |
|                                   |     private:                      |
|                                   |         void dispatchVsy          |
|                                   | nc(nsecs_t timestamp, PhysicalDis |
|                                   | playId displayId, uint32_t count, |
|                                   |                 VsyncEv           |
|                                   | entData vsyncEventData) override; |
|                                   |                                   |
|                                   |      void dispatchHotplug(nsecs_t |
|                                   |  timestamp, PhysicalDisplayId dis |
|                                   | playId, bool connected) override; |
|                                   |     };                            |
+-----------------------------------+-----------------------------------+

**4）Native 收发通道：`DisplayEventReceiver` + `BitTube`（C++）**

+-----------------------------------+-----------------------------------+
|     1                             |     // frameworks/native/         |
|     2                             | libs/gui/DisplayEventReceiver.cpp |
|     3                             |     DisplayEventReceiver::Di      |
|     4                             | splayEventReceiver(gui::ISurfaceC |
|     5                             | omposer::VsyncSource vsyncSource, |
|     6                             |             EventRegi             |
|     7                             | strationFlags eventRegistration,  |
|     8                             | const sp<IBinder>& layerHandle) { |
|     9                             |         sf->createDisplay         |
|     10                            | EventConnection(vsyncSource, ..., |
|     11                            |  layerHandle, &mEventConnection); |
|     12                            |         mDataChannel =            |
|     13                            | std::make_unique<gui::BitTube>(); |
|     14                            |         mEventConnection->stealRe |
|     15                            | ceiveChannel(mDataChannel.get()); |
|     16                            |     }                             |
|                                   |                                   |
|                                   |     status_t DisplayEv            |
|                                   | entReceiver::requestNextVsync() { |
|                                   |         mEve                      |
|                                   | ntConnection->requestNextVsync(); |
|                                   |         return NO_ERROR;          |
|                                   |     }                             |
|                                   |                                   |
|                                   |     ssiz                          |
|                                   | e_t DisplayEventReceiver::getEven |
|                                   | ts(Event* events, size_t count) { |
|                                   |         re                        |
|                                   | turn gui::BitTube::recvObjects(mD |
|                                   | ataChannel.get(), events, count); |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

**5）SurfaceFlinger 调度与分发（C++）**

+-----------------------------------+-----------------------------------+
|     1                             |     // f                          |
|     2                             | rameworks/native/services/surface |
|     3                             | flinger/Scheduler/VSyncDispatch.h |
|                                   |     virtual std::o                |
|                                   | ptional<ScheduleResult> schedule( |
|                                   |                                   |
|                                   |           CallbackToken token, Sc |
|                                   | heduleTiming scheduleTiming) = 0; |
+-----------------------------------+-----------------------------------+

+-----------------------------------+-----------------------------------+
|     1                             |     // f                          |
|     2                             | rameworks/native/services/surface |
|     3                             | flinger/Scheduler/EventThread.cpp |
|     4                             |     void EventThread::o           |
|     5                             | nVsync(nsecs_t vsyncTime, nsecs_t |
|     6                             |  wakeupTime, nsecs_t readyTime) { |
|     7                             |         mPendingEvents.push_      |
|     8                             | back(makeVSync(mVsyncSchedule->ge |
|     9                             | tPhysicalDisplayId(), wakeupTime, |
|     10                            |                 ++mVSyncStat      |
|     11                            | e->count, vsyncTime, readyTime)); |
|     12                            |     }                             |
|     13                            |                                   |
|     14                            |                                   |
|     15                            | void EventThread::threadMain(std: |
|     16                            | :unique_lock<std::mutex>& lock) { |
|                                   |         ...                       |
|                                   |         const auto scheduleResul  |
|                                   | t = mVsyncRegistration.schedule({ |
|                                   |                 .workDurati       |
|                                   | on = mWorkDuration.get().count(), |
|                                   |                 .readyD           |
|                                   | uration = mReadyDuration.count(), |
|                                   |                 .lastVsy          |
|                                   | nc = mLastVsyncCallbackTime.ns(), |
|                                   |                                   |
|                                   |               .committedVsyncOpt  |
|                                   | = mLastCommittedVsyncTime.ns()}); |
|                                   |                                   |
|                                   | LOG_ALWAYS_FATAL_IF(!scheduleResu |
|                                   | lt, "Error scheduling callback"); |
|                                   |         ...                       |
|                                   |     }                             |
+-----------------------------------+-----------------------------------+

### [](#关键时序点分析 "关键时序点分析"){.headerlink}关键时序点分析

沿着上述代码流程，可以梳理出完整的时序链路：

1.  **HWC 硬件 Vsync 或软件预测节拍** → SurfaceFlinger Scheduler
    更新显示时序
2.  **Scheduler 计算唤醒窗口** → `VSyncDispatch::schedule(...)`
3.  **EventThread 生成/派发事件** → 写入
    `DisplayEventReceiver::Event`（通过 `BitTube`）
4.  **App 侧 Native 收到事件** →
    `DisplayEventDispatcher::dispatchVsync(...)`
5.  **Java `FrameDisplayEventReceiver` 回调** → 异步消息切到 Looper 队列
6.  **`Choreographer#doFrame(...)` 执行** →
    Input/Animation/Traversal/Commit

各环节的职责和优化点不同，理解完整流程有助于在 Perfetto 中分析 Vsync
相关性能问题。

### [](#FrameTimeline "FrameTimeline"){.headerlink}FrameTimeline {#FrameTimeline}

App 和 SurfaceFlinger 都有 FrameTimeline

![image-20250918223035361](/images/Android-Perfetto-08-Vsync/image-20250918223035361.webp)

-   **轨道**：`Expected Timeline`、`Actual Timeline`
-   **PresentType/JankType**：
-   PresentType 指示本帧呈现方式（例如 On-time、Late），JankType
    指示卡顿类型来源
-   常见
    JankType：`AppDeadlineMissed`、`BufferStuffing`、`SfCpuDeadlineMissed`、`SfGpuDeadlineMissed`
    等
-   **操作步骤（Perfetto UI）**：

1.  在应用进程选择目标 `Surface/Layer` 或使用 FrameToken 过滤
2.  对齐 Expected 与 Actual，查看偏移与颜色编码
3.  向上钻取：`Choreographer#doFrame`、`RenderThread`、`queueBuffer`、`acquire/present_fence`

-   **误判规避**：
-   仅凭 `doFrame` 时长判断掉帧不可靠；以 FrameTimeline 的
    PresentType/JankType 为准
-   多缓冲可能掩盖单帧超时，需要看连续帧与 Buffer 可用性

### [](#刷新率-x2F-显示模式-x2F-VRR-对-Vsync-与-Offset-x2F-预测的影响 "刷新率/显示模式/VRR 对 Vsync 与 Offset/预测的影响"){.headerlink}刷新率/显示模式/VRR 对 Vsync 与 Offset/预测的影响 {#刷新率-x2F-显示模式-x2F-VRR-对-Vsync-与-Offset-x2F-预测的影响}

-   **模式切换**：刷新率变更会重新配置 `VsyncConfiguration`，影响 app/sf
    Offset 与预测模型；
-   Perfetto：查 `display mode change` 事件与随后的 `vsync` 间隔变化
-   **VRR（可变刷新率）**：目标周期不恒定，软件预测更依赖 present\_fence
    反馈校准；
-   Perfetto：观察 `vsync` 间隔分布与 `present_fence` 偏差
-   **多显示/外接显示**：硬件层可按 `physicalDisplayId` 上报
    vsync；但应用侧 Choreographer 通常仍以内屏/pacesetter
    时序为主（实现细节随版本演进）。分析时先确认你看的到底是 HWC/SF
    轨道，还是 app 轨道；
-   Android 12--17 分析时，仍要区分框架/HWC 的 per-display VSYNC
    能力与应用侧 `Choreographer` 的请求路径；后者可能受 internal display
    / pacesetter 实现约束，以目标系统分支和设备 Trace 为准
-   Perfetto：按显示 ID 过滤相关 Counter/Slice

### [](#Perfetto-实战-Checklist（建议按序查看） "Perfetto 实战 Checklist（建议按序查看）"){.headerlink}Perfetto 实战 Checklist（建议按序查看） {#Perfetto-实战-Checklist（建议按序查看）}

1.  **Vsync 信号与周期**

-   `vsync-app / vsync-sf / vsync-appSf` 间隔是否稳定（60/90/120Hz
    对应周期）
-   是否存在异常密集/稀疏的 Vsync（预测抖动）\
    ![image-20250918223148748](/images/Android-Perfetto-08-Vsync/image-20250918223148748.webp)

2.  **Vsync 相位差配置**

-   `VsyncWorkDuration` 是否符合机型预期的 app/sf Offset
-   app 与 sf 的先后是否匹配"先绘制后合成"的策略\
    ![image-20250918222707300](/images/Android-Perfetto-08-Vsync/image-20250918222707300.webp)

3.  **FrameTimeline 判读**

-   先看 `PresentType`，再看 `JankType`；确认是 app 还是 SF/GPU 侧问题
-   选择目标 Surface/FrameToken 定位具体帧\
    ![image-20250918223220718](/images/Android-Perfetto-08-Vsync/image-20250918223220718.webp)

4.  **应用主线程与渲染线程**

-   `Choreographer#doFrame` 各阶段耗时（Input/Animation/Traversal）
-   `RenderThread` 的 `syncAndDrawFrame/DrawFrame` 耗时是否异常\
    ![image-20250918223340940](/images/Android-Perfetto-08-Vsync/image-20250918223340940.webp)

5.  **BufferQueue 与 Fence**

-   生产者：RenderThread `queueBuffer` 之后，Buffer 进入可消费队列；但
    SF 是否能立刻 latch 还要看 `acquire fence`。`present fence`
    主要用于确认该帧实际送显完成时间。新版本在特定策略下可对 unsignaled
    buffer 先推进，再在需要时等待 fence。\
    ![image-20250918224205093](/images/Android-Perfetto-08-Vsync/image-20250918224205093.webp)
-   消费者 SF 与 BufferTX：SF 在有待处理更新的合成节拍会尝试为目标 layer
    取最新可用 Buffer。若某 layer 的 BufferTX 为 0，通常表示该 layer
    暂无待处理 Buffer；静态页面属正常情况，只有本应持续更新却没有新
    Buffer 时才需排查 App 侧。它不代表 SF 全局"停止合成"。\
    ![image-20250918223441117](/images/Android-Perfetto-08-Vsync/image-20250918223441117.webp)

6.  **合成策略与显示**

-   SF 是否频繁走 ClientComposition；HWC validate/present 是否异常
-   多显示/模式切换/VRR 时是否伴随明显预测偏差\
    ![image-20250918223517315](/images/Android-Perfetto-08-Vsync/image-20250918223517315.webp)

7.  **资源与其他干扰**

-   CPU 竞争（大核占用）、GPU 忙、IO/内存抖动（GC/compaction）
-   其他前台应用/系统服务是否占用关键资源\
    ![image-20250918223532482](/images/Android-Perfetto-08-Vsync/image-20250918223532482.webp)

[]{#refs}

[](#参考文档 "参考文档"){.headerlink}参考文档
---------------------------------------------

1.  [Android Graphics
    Architecture](https://source.android.com/docs/core/graphics)
2.  [VSYNC Implementation
    Guide](https://source.android.com/docs/core/graphics/implement-vsync)
3.  [Frame
    Pacing](https://source.android.com/docs/core/graphics/frame-pacing)
4.  [Perfetto Documentation](https://perfetto.dev/docs/)
5.  [Android Perfetto 系列 5：Android App 基于 Choreographer
    的渲染流程](https://www.androidperformance.com/2025/03/26/Android-Perfetto-05-Chorergrapher/)
6.  [Android Perfetto 系列 6：为什么是
    120Hz？高刷新率的优势与挑战](https://www.androidperformance.com/2025/04/26/Android-Perfetto-06-Why-120Hz/)
7.  [Vsync offset
    相关技术分析](https://cloud.tencent.com/developer/article/1905184)
8.  [Android
    13/14高版本SurfaceFlinger出现VSYNC-app/VSYNC-appSf/VSYNC-sf剖析](https://blog.csdn.net/learnframework/article/details/133964332)
9.  [AOSP -
    Choreographer.java（main）](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/Choreographer.java)
10. [AOSP -
    android\_view\_DisplayEventReceiver.cpp（main）](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/android_view_DisplayEventReceiver.cpp)
11. [AOSP -
    DisplayEventDispatcher.h（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/gui/include/gui/DisplayEventDispatcher.h)
12. [AOSP -
    DisplayEventReceiver.cpp（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/gui/DisplayEventReceiver.cpp)
13. [AOSP -
    VSyncDispatch.h（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/surfaceflinger/Scheduler/VSyncDispatch.h)
14. [AOSP -
    EventThread.cpp（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/surfaceflinger/Scheduler/EventThread.cpp)
15. [Android
    Multi-display（官方文档）](https://source.android.com/docs/core/display/multi_display/three-hardware-displays)
16. [AOSP -
    VsyncConfiguration.h（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/surfaceflinger/Scheduler/VsyncConfiguration.h)
17. [AOSP -
    DisplayEventDispatcher.cpp（main）](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/gui/DisplayEventDispatcher.cpp)
18. [Unsignaled buffer
    latch（官方文档）](https://source.android.com/docs/core/graphics/unsignaled-buffer-latch)

[]{#about}

[](#关于我-amp-amp-博客 "关于我 && 博客"){.headerlink}关于我 && 博客 {#关于我-amp-amp-博客}
====================================================================

下面是个人的介绍和相关的链接，期望与同行的各位多多交流，三人行，则必有我师!

1.  [博主个人介绍](https://www.androidperformance.com/about/)
    ：里面有个人的微信和微信群链接。
2.  [本博客内容导航](https://www.androidperformance.com/2019/12/01/BlogMap/)
    ：个人博客内容的一个导航。
3.  [个人整理和搜集的优秀博客文章 - Android
    性能优化必知必会](https://androidperformance.com/2018/05/07/Android-performance-optimization-skills-and-tools/)
    ：欢迎大家自荐和推荐 （微信私聊即可）
4.  [Android性能优化知识星球](https://www.androidperformance.com/2023/12/30/the-performance/)
    ： 欢迎加入，多谢支持～

> **一个人可以走的更快 , 一群人可以走的更远**

![微信扫一扫](/images/WechatIMG581.webp)

::: {.license-wrapper}
Author：[Gracker](https://androidperformance.com)

Link：<https://androidperformance.com/2025/08/05/Android-Perfetto-08-Vsync/>

Publish date：[August 5th 2025, 10:15:30
am](https://androidperformance.com/2025/08/05/Android-Perfetto-08-Vsync/)

Update date：[October 6th 2026, 7:48:39
pm](https://androidperformance.com/2025/08/05/Android-Perfetto-08-Vsync/)

License：本文采用[知识共享署名-非商业性使用 4.0
国际许可协议](http://creativecommons.org/licenses/by-nc/4.0/)进行许可
:::

-   ::: {.nextSlogan}
    Next Post
    :::

    [](/2025/11/12/Android-Perfetto-09-CPU/ "Android Perfetto 系列 9：CPU 信息解读")
    ::: {.nextTitle}
    Android Perfetto 系列 9：CPU 信息解读
    :::

-   ::: {.prevSlogan}
    Previous Post
    :::

    [](/2025/08/02/Android-Perfetto-07-MainThread-And-RenderThread/ "Android Perfetto 系列 7：MainThread 和 RenderThread 解读")
    ::: {.prevTitle}
    Android Perfetto 系列 7：MainThread 和 RenderThread 解读
    :::

::: {.post-comment}
:::
:::
:::

::: {.social}
[](mailto:Dreamtale.jg@gmail.com "email"){.iconfont-archer .email}
[](https://github.com/gracker "github"){.iconfont-archer .github} [
![](/assets/Wechat.jpeg){.profile-qr} ]{.iconfont-archer .wechat
title="wechat"} [ ]{.iconfont-archer .qq title="qq: 553000664"}
[](https://weibo.com/gracker520 "weibo"){.iconfont-archer .weibo}
[](https://www.zhihu.com/people/gracker "zhihu"){.iconfont-archer
.zhihu} [](https://twitter.com/Gracker_Gao "twitter"){.iconfont-archer
.twitter}
[](https://juejin.im/user/557b0590e4b052e031453d76 "juejin"){.iconfont-archer
.juejin} [](/atom.xml "rss"){.iconfont-archer .rss}
:::

::: {.copyright}
[Powered by [Hexo](https://hexo.io/)]{#hexo-power}[]{.iconfont-archer
.power}[theme
[Archer](https://github.com/fi3ework/hexo-theme-archer)]{#theme-info}
:::

::: {.website-approve}
[ [蜀 ICP 备 20016190 号](https://beian.miit.gov.cn/) ]{#icp-approve
.icp-approve}
:::
:::

::: {.toc-wrapper .toc-wrapper-loding style="top:50vh;"}
::: {.toc-catalog}
[]{.iconfont-archer .catalog-icon}CATALOG
:::

1.  [[1.]{.toc-number}
    [本文目录]{.toc-text}](#%E6%9C%AC%E6%96%87%E7%9B%AE%E5%BD%95){.toc-link}

[[]{.toc-number}
[系列文章目录]{.toc-text}](#%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E7%9B%AE%E5%BD%95){.toc-link}

1.  [[1.]{.toc-number} [什么是
    Vsync]{.toc-text}](#%E4%BB%80%E4%B9%88%E6%98%AF-Vsync){.toc-link}
    1.  [[1.1.]{.toc-number} [Vsync
        解决什么问题？]{.toc-text}](#Vsync-%E8%A7%A3%E5%86%B3%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98%EF%BC%9F){.toc-link}
2.  [[2.]{.toc-number} [Android 中 Vsync
    的基本工作原理]{.toc-text}](#Android-%E4%B8%AD-Vsync-%E7%9A%84%E5%9F%BA%E6%9C%AC%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86){.toc-link}
    1.  [[2.1.]{.toc-number} [Vsync
        信号的分层架构]{.toc-text}](#Vsync-%E4%BF%A1%E5%8F%B7%E7%9A%84%E5%88%86%E5%B1%82%E6%9E%B6%E6%9E%84){.toc-link}
3.  [[3.]{.toc-number} [在 Perfetto 中观察
    Vsync]{.toc-text}](#%E5%9C%A8-Perfetto-%E4%B8%AD%E8%A7%82%E5%AF%9F-Vsync){.toc-link}
4.  [[4.]{.toc-number} [Android App 每一帧是如何基于 Vsync
    工作的]{.toc-text}](#Android-App-%E6%AF%8F%E4%B8%80%E5%B8%A7%E6%98%AF%E5%A6%82%E4%BD%95%E5%9F%BA%E4%BA%8E-Vsync-%E5%B7%A5%E4%BD%9C%E7%9A%84){.toc-link}
    1.  [[4.1.]{.toc-number}
        [流程总览（按顺序）]{.toc-text}](#%E6%B5%81%E7%A8%8B%E6%80%BB%E8%A7%88%EF%BC%88%E6%8C%89%E9%A1%BA%E5%BA%8F%EF%BC%89){.toc-link}
    2.  [[4.2.]{.toc-number} [App 什么时候会申请 Vsync
        信号]{.toc-text}](#App-%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E4%BC%9A%E7%94%B3%E8%AF%B7-Vsync-%E4%BF%A1%E5%8F%B7){.toc-link}
    3.  [[4.3.]{.toc-number} [App 申请 Vsync
        的完整流程]{.toc-text}](#App-%E7%94%B3%E8%AF%B7-Vsync-%E7%9A%84%E5%AE%8C%E6%95%B4%E6%B5%81%E7%A8%8B){.toc-link}
    4.  [[4.4.]{.toc-number} [主线程如何监听 Vsync
        信号]{.toc-text}](#%E4%B8%BB%E7%BA%BF%E7%A8%8B%E5%A6%82%E4%BD%95%E7%9B%91%E5%90%AC-Vsync-%E4%BF%A1%E5%8F%B7){.toc-link}
    5.  [[4.5.]{.toc-number}
        [几个遗留问题]{.toc-text}](#%E5%87%A0%E4%B8%AA%E9%81%97%E7%95%99%E9%97%AE%E9%A2%98){.toc-link}
    6.  [[4.6.]{.toc-number} [Vsync Offset / WorkDuration
        的技术实现]{.toc-text}](#Vsync-Offset-x2F-WorkDuration-%E7%9A%84%E6%8A%80%E6%9C%AF%E5%AE%9E%E7%8E%B0){.toc-link}
    7.  [[4.7.]{.toc-number}
        [相位差的简化时序示意]{.toc-text}](#%E7%9B%B8%E4%BD%8D%E5%B7%AE%E7%9A%84%E7%AE%80%E5%8C%96%E6%97%B6%E5%BA%8F%E7%A4%BA%E6%84%8F){.toc-link}
5.  [[5.]{.toc-number} [Vsync
    信号的完整代码流程]{.toc-text}](#Vsync-%E4%BF%A1%E5%8F%B7%E7%9A%84%E5%AE%8C%E6%95%B4%E4%BB%A3%E7%A0%81%E6%B5%81%E7%A8%8B){.toc-link}
    1.  [[5.1.]{.toc-number} [按 AOSP main
        分支对齐的关键代码（精简摘录）]{.toc-text}](#%E6%8C%89-AOSP-main-%E5%88%86%E6%94%AF%E5%AF%B9%E9%BD%90%E7%9A%84%E5%85%B3%E9%94%AE%E4%BB%A3%E7%A0%81%EF%BC%88%E7%B2%BE%E7%AE%80%E6%91%98%E5%BD%95%EF%BC%89){.toc-link}
    2.  [[5.2.]{.toc-number}
        [关键时序点分析]{.toc-text}](#%E5%85%B3%E9%94%AE%E6%97%B6%E5%BA%8F%E7%82%B9%E5%88%86%E6%9E%90){.toc-link}
    3.  [[5.3.]{.toc-number}
        [FrameTimeline]{.toc-text}](#FrameTimeline){.toc-link}
    4.  [[5.4.]{.toc-number} [刷新率/显示模式/VRR 对 Vsync 与
        Offset/预测的影响]{.toc-text}](#%E5%88%B7%E6%96%B0%E7%8E%87-x2F-%E6%98%BE%E7%A4%BA%E6%A8%A1%E5%BC%8F-x2F-VRR-%E5%AF%B9-Vsync-%E4%B8%8E-Offset-x2F-%E9%A2%84%E6%B5%8B%E7%9A%84%E5%BD%B1%E5%93%8D){.toc-link}
    5.  [[5.5.]{.toc-number} [Perfetto 实战
        Checklist（建议按序查看）]{.toc-text}](#Perfetto-%E5%AE%9E%E6%88%98-Checklist%EF%BC%88%E5%BB%BA%E8%AE%AE%E6%8C%89%E5%BA%8F%E6%9F%A5%E7%9C%8B%EF%BC%89){.toc-link}
6.  [[6.]{.toc-number}
    [参考文档]{.toc-text}](#%E5%8F%82%E8%80%83%E6%96%87%E6%A1%A3){.toc-link}

[[]{.toc-number} [关于我 &&
博客]{.toc-text}](#%E5%85%B3%E4%BA%8E%E6%88%91-amp-amp-%E5%8D%9A%E5%AE%A2){.toc-link}
:::

::: {.sidebar .sidebar-hide}
-   []{.iconfont-archer}[Archive]{.tab-name}
-   []{.iconfont-archer}[Tag]{.tab-name}
-   []{.iconfont-archer}[Cate]{.tab-name}

::: {.sidebar-content .sidebar-content-show-archive}
::: {.sidebar-panel-archives}
::: {.total-and-search}
::: {.total-archive}
Total : 140
:::
:::

::: {.post-archive}
::: {.archive-year}
2026
:::

-   [10/06]{.archive-post-date} [SmartPerfetto v1.15.1
    更新说明](/2026/10/06/SmartPerfetto-v1.15.1-Release-Notes/){.archive-post-title}
-   [09/19]{.archive-post-date} [SmartPerfetto v1.12.0
    更新说明](/2026/09/19/SmartPerfetto-v1.12.0-Release-Notes/){.archive-post-title}
-   [09/19]{.archive-post-date} [耗时大半年，开源一本 Android
    电子书《Android
    技术内幕：系统机制、性能优化与工具实战》](/2026/09/19/AIW-Open-Source-Android-Internals-Wiki/){.archive-post-title}
-   [08/21]{.archive-post-date} [SmartPerfetto v1.7.0
    更新说明](/2026/08/21/SmartPerfetto-Five-Week-Update/){.archive-post-title}
-   [07/17]{.archive-post-date} [SmartPerfetto v1.1.1
    更新说明](/2026/07/17/SmartPerfetto-Six-Week-Update/){.archive-post-title}
-   [06/04]{.archive-post-date} [SmartPerfetto v1.0.28
    更新说明](/2026/06/04/SmartPerfetto-Three-Week-Update/){.archive-post-title}
-   [05/17]{.archive-post-date} [SmartPerfetto v1.0.7
    更新说明](/2026/05/17/SmartPerfetto-Two-Week-Update/){.archive-post-title}
-   [05/05]{.archive-post-date}
    [我做了什么](/2026/05/05/What-I-Am-Building/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列
    18：响应速度实战，从输入事件到首帧反馈](/2026/05/04/Android-Perfetto-18-Input-Response-Latency/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列
    17：专项场景自动化、问题清单与平台体系](/2026/05/04/Android-Perfetto-17-Engineering-Domain-And-Platform-Tracing/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列 16：GPU、Power
    Counters
    与硬件瓶颈分析](/2026/05/04/Android-Perfetto-16-GPU-Power-Counters/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列 15：Boot
    Trace、长时间现场 Trace
    与偶发问题抓取](/2026/05/04/Android-Perfetto-15-Boot-And-Long-Running-Field-Tracing/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列 14：heapprofd
    与内存
    Profiling](/2026/05/04/Android-Perfetto-14-Native-Heap-Heapprofd/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列 13：Perfetto
    SDK、Track Event 与 App 现场
    Trace](/2026/05/04/Android-Perfetto-13-Track-Event-SDK/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列 12：Trace
    数据流与丢失排查](/2026/05/04/Android-Perfetto-12-Trace-Dataflow-And-Loss/){.archive-post-title}
-   [05/04]{.archive-post-date} [Android Perfetto 系列
    11：PerfettoSQL、Trace Processor
    与回归检测](/2026/05/04/Android-Perfetto-11-PerfettoSQL-TraceProcessor-Automation/){.archive-post-title}
-   [04/29]{.archive-post-date} [SmartPerfetto 开源：面向 Android Trace
    分析的 Perfetto AI
    Assistant](/2026/04/29/SmartPerfetto-Open-Source-Perfetto-AI-Assistant/){.archive-post-title}
-   [04/28]{.archive-post-date} [万字长文推演：手机不再从 App
    开始，Agent OS
    如何接管任务入口](/2026/04/28/Agent-OS-Task-Entry/){.archive-post-title}
-   [04/10]{.archive-post-date} [SmartPerfetto 架构文章 Q&A：8
    个深度技术问答](/2026/04/10/SmartPerfetto-Architecture-Deep-Dive-QA/){.archive-post-title}
-   [04/10]{.archive-post-date} [从 Trace 到洞察：SmartPerfetto AI Agent
    的 Harness Engineering
    实战](/2026/04/10/SmartPerfetto-Architecture-Deep-Dive/){.archive-post-title}
-   [03/10]{.archive-post-date} [OpenClaw 常见问题解答：Token
    消耗、能干什么、本地模型、隐私安全、使用体验](/2026/03/10/OpenClaw-FAQ/){.archive-post-title}
-   [03/08]{.archive-post-date} [我把 OpenClaw
    跑在本地三周后，发现它根本不是聊天机器人](/2026/03/08/OpenClaw-Local-Practice/){.archive-post-title}

::: {.archive-year}
2025
:::

-   [11/16]{.archive-post-date} [Android Perfetto 系列 10：Binder
    调度与锁竞争](/2025/11/16/Android-Perfetto-10-Binder/){.archive-post-title}
-   [11/12]{.archive-post-date} [Android Perfetto 系列 9：CPU
    信息解读](/2025/11/12/Android-Perfetto-09-CPU/){.archive-post-title}
-   [08/05]{.archive-post-date} [Android Perfetto 系列 8：深入理解 Vsync
    机制与性能分析](/2025/08/05/Android-Perfetto-08-Vsync/){.archive-post-title}
-   [08/02]{.archive-post-date} [Android Perfetto 系列 7：MainThread 和
    RenderThread
    解读](/2025/08/02/Android-Perfetto-07-MainThread-And-RenderThread/){.archive-post-title}
-   [04/26]{.archive-post-date} [Android Perfetto 系列 6：为什么是
    120Hz？高刷新率的优势与挑战](/2025/04/26/Android-Perfetto-06-Why-120Hz/){.archive-post-title}
-   [03/26]{.archive-post-date} [Android Perfetto 系列 5：Android App
    基于 Choreographer
    的渲染流程](/2025/03/26/Android-Perfetto-05-Chorergrapher/){.archive-post-title}
-   [02/08]{.archive-post-date} [Android Perfetto 系列
    4：使用命令行在本地打开超大
    Trace](/2025/02/08/Android-Perfetto-04-Open-Big-Trace-With-Command-Line/){.archive-post-title}
-   [02/08]{.archive-post-date} [Android ANR 系列 3 ：ANR
    案例分享](/2025/02/08/Android-ANR-03-ANR-Case-Share/){.archive-post-title}
-   [02/08]{.archive-post-date} [Android ANR 系列 2 ：ANR 分析套路和关键
    Log
    介绍](/2025/02/08/Android-ANR-02-How-to-analysis-ANR/){.archive-post-title}
-   [02/08]{.archive-post-date} [Android ANR 系列 1 ：理解 Android ANR
    设计思想](/2025/02/08/Android-ANR-01-ANR-Design/){.archive-post-title}

::: {.archive-year}
2024
:::

-   [05/21]{.archive-post-date} [Android Perfetto 系列 3：熟悉 Perfetto
    View](/2024/05/21/Android-Perfetto-03-how-to-analysis-perfetto/){.archive-post-title}
-   [05/21]{.archive-post-date} [Android Perfetto 系列 2：Perfetto Trace
    抓取](/2024/05/21/Android-Perfetto-02-how-to-get-perfetto/){.archive-post-title}
-   [05/21]{.archive-post-date} [Android Perfetto 系列 1：Perfetto
    工具简介](/2024/05/21/Android-Perfetto-01-What-is-perfetto/){.archive-post-title}
-   [03/27]{.archive-post-date} [Android Perfetto
    系列目录](/2024/03/27/Android-Perfetto-101/){.archive-post-title}
-   [01/01]{.archive-post-date} [2023
    年的方方面面](/2024/01/01/2023-review/){.archive-post-title}

::: {.archive-year}
2023
:::

-   [10/15]{.archive-post-date} [The Performance Design Of
    OS](/2023/10/15/the-performance-design-of-os-en/){.archive-post-title}
-   [08/21]{.archive-post-date} [OS
    设计之性能设计](/2023/08/21/the-performance-design-of-os/){.archive-post-title}
-   [05/14]{.archive-post-date} [当 App
    有了系统权限，真的可以为所欲为？](/2023/05/14/bad-android-app-with-system-permissions/){.archive-post-title}

::: {.archive-year}
2022
:::

-   [03/27]{.archive-post-date} [The Performance 星球茶话会 -
    第一期](/2022/03/27/the-performance-tea-part-01/){.archive-post-title}
-   [03/13]{.archive-post-date} [Systrace 线程 CPU 运行状态分析技巧 -
    Sleep 和 Uninterruptible Sleep
    篇](/2022/03/13/android-systrace-cpu-state-sleep/){.archive-post-title}
-   [03/13]{.archive-post-date} [Systrace 线程 CPU 运行状态分析技巧 -
    Running
    篇](/2022/03/13/android-systrace-cpu-state-running/){.archive-post-title}
-   [01/21]{.archive-post-date} [Systrace 线程 CPU 运行状态分析技巧 -
    Runnable
    篇](/2022/01/21/android-systrace-cpu-state-runnable/){.archive-post-title}
-   [01/07]{.archive-post-date} [Techniques, Philosophy, and Tools for
    Android Performance
    Optimization](/2022/01/07/Techniques-Philosophy-and-Tools-for-Android-Performance-Optimization/){.archive-post-title}
-   [01/07]{.archive-post-date} [Android
    性能优化的术、道、器](/2022/01/07/The-Performace-1-Performance-Tools/){.archive-post-title}
-   [01/03]{.archive-post-date} [回顾
    2021](/2022/01/03/2021-Review/){.archive-post-title}

::: {.archive-year}
2021
:::

-   [10/27]{.archive-post-date} [一本讲 Android
    流畅性的书，应该有什么内容？](/2021/10/27/if-i-write-a-book-about-performance/){.archive-post-title}
-   [10/26]{.archive-post-date} [Android 系统开发系列（1）：Android 12
    源代码下载、编译和刷机](/2021/10/26/build-android-12/){.archive-post-title}
-   [09/13]{.archive-post-date} [Android Systrace 响应速度实战 3
    ：响应速度延伸知识](/2021/09/13/android-systrace-Responsiveness-in-action-3/){.archive-post-title}
-   [09/13]{.archive-post-date} [Android Systrace 响应速度实战 2
    ：响应速度实战分析-以启动速度为例](/2021/09/13/android-systrace-Responsiveness-in-action-2/){.archive-post-title}
-   [09/13]{.archive-post-date} [Android Systrace 响应速度实战 1
    ：了解响应速度原理](/2021/09/13/android-systrace-Responsiveness-in-action-1/){.archive-post-title}
-   [04/24]{.archive-post-date} [Android Systrace 流畅性实战 3
    ：卡顿分析过程中的一些疑问](/2021/04/24/android-systrace-smooth-in-action-3/){.archive-post-title}
-   [04/24]{.archive-post-date} [Android Systrace 流畅性实战 2
    ：案例分析 - MIUI
    桌面滑动卡顿分析](/2021/04/24/android-systrace-smooth-in-action-2/){.archive-post-title}
-   [04/24]{.archive-post-date} [Android Systrace 流畅性实战 1
    ：了解卡顿原理](/2021/04/24/android-systrace-smooth-in-action-1/){.archive-post-title}

::: {.archive-year}
2020
:::

-   [08/20]{.archive-post-date}
    [华为手机刷微博体验更好？技术角度的一些分析和思考](/2020/08/20/weibo-imageload-opt-on-huawei/){.archive-post-title}
-   [05/26]{.archive-post-date} [一个「闰」字引发的事故 -
    三星系统重启分析](/2020/05/26/samsung_crash/){.archive-post-title}
-   [05/07]{.archive-post-date} [Android App
    链式唤醒分析](/2020/05/07/Android-App-Chain-Wakeup/){.archive-post-title}
-   [02/14]{.archive-post-date} [Android Systrace 基础知识 -
    SurfaceFlinger
    解读](/2020/02/14/Android-Systrace-SurfaceFlinger/){.archive-post-title}
-   [02/03]{.archive-post-date} [Android 开发者学习路线(2020
    版本)](/2020/02/03/android-development-learning-path-2020-edition/){.archive-post-title}
-   [01/28]{.archive-post-date} [我的 2020
    年读书单](/2020/01/28/2020-read/){.archive-post-title}

::: {.archive-year}
2019
:::

-   [12/21]{.archive-post-date} [Android Systrace 基础知识 - CPU Info
    解读](/2019/12/21/Android-Systrace-CPU/){.archive-post-title}
-   [12/15]{.archive-post-date} [Android Systrace 基础知识 - Triple
    Buffer
    解读](/2019/12/15/Android-Systrace-Triple-Buffer/){.archive-post-title}
-   [12/06]{.archive-post-date} [Android Systrace 基础知识 - Binder
    和锁竞争解读](/2019/12/06/Android-Systrace-Binder/){.archive-post-title}
-   [12/01]{.archive-post-date} [「置顶」博客文章目录 +
    个人项目](/2019/12/01/BlogMap/){.archive-post-title}
-   [12/01]{.archive-post-date} [Android Systrace 基础知识 - Vsync
    解读](/2019/12/01/Android-Systrace-Vsync/){.archive-post-title}
-   [11/18]{.archive-post-date} [Android App
    启动优化全记录](/2019/11/18/Android-App-Lunch-Optimize/){.archive-post-title}
-   [11/06]{.archive-post-date} [Android Systrace 基础知识 - MainThread
    和 RenderThread
    解读](/2019/11/06/Android-Systrace-MainThread-And-RenderThread/){.archive-post-title}
-   [11/04]{.archive-post-date} [Android Systrace 基础知识 - Input
    解读](/2019/11/04/Android-Systrace-Input/){.archive-post-title}
-   [10/24]{.archive-post-date} [Android
    中的"后台无效动画"行为分析](/2019/10/24/Android-Background-Animation/){.archive-post-title}
-   [10/22]{.archive-post-date} [Android 基于 Choreographer
    的渲染机制详解](/2019/10/22/Android-Choreographer/){.archive-post-title}
-   [09/18]{.archive-post-date} [Android 中的卡顿丢帧原因概述 -
    低内存篇](/2019/09/18/Android-Jank-Due-To-Low-Memory/){.archive-post-title}
-   [09/17]{.archive-post-date} [Android
    桌面被杀问题分析案例](/2019/09/17/Android-Kill-Background-App-Debug/){.archive-post-title}
-   [09/05]{.archive-post-date} [Android 中的卡顿丢帧原因概述 -
    应用篇](/2019/09/05/Android-Jank-Due-To-App/){.archive-post-title}
-   [09/05]{.archive-post-date} [Android 中的卡顿丢帧原因概述 -
    系统篇](/2019/09/05/Android-Jank-Due-To-System/){.archive-post-title}
-   [09/05]{.archive-post-date} [Android 中的卡顿丢帧原因概述 -
    方法论](/2019/09/05/Android-Jank-Debug/){.archive-post-title}
-   [09/01]{.archive-post-date} [Android 中的 Activity Launch Mode
    详解](/2019/09/01/Android-Activity-Lunch-Mode/){.archive-post-title}
-   [07/27]{.archive-post-date} [Android 中的 Hardware Layer
    详解](/2019/07/27/Android-Hardware-Layer/){.archive-post-title}
-   [07/23]{.archive-post-date} [Android Systrace 基础知识 \-- 分析
    Systrace
    预备知识](/2019/07/23/Android-Systrace-Pre/){.archive-post-title}
-   [06/29]{.archive-post-date} [Android Systrace 基础知识 -
    SystemServer
    解读](/2019/06/29/Android-Systrace-SystemServer/){.archive-post-title}
-   [05/28]{.archive-post-date} [Android Systrace 基础知识 \-- Systrace
    简介](/2019/05/28/Android-Systrace-About/){.archive-post-title}
-   [05/27]{.archive-post-date} [Android Systrace 基础知识 \-- Why 60
    fps ？](/2019/05/27/why-60-fps/){.archive-post-title}
-   [05/26]{.archive-post-date} [Android Systrace \--
    系列文章目录](/2019/05/26/Android_Systrace_0/){.archive-post-title}
-   [05/15]{.archive-post-date} [Android 新的流畅体验，90Hz
    漫谈](/2019/05/15/90hz-on-android/){.archive-post-title}
-   [04/07]{.archive-post-date} [利器 -
    高效工具推荐](/2019/04/07/liqi/){.archive-post-title}
-   [01/21]{.archive-post-date} [Android
    无障碍服务导致的整机卡顿案例分析](/2019/01/21/android-performance-case-jank-accessbility/){.archive-post-title}
-   [01/12]{.archive-post-date} [2018 年度好物推荐 -
    给辛勤工作的自己一点奖励](/2019/01/12/recommend-of-2018/){.archive-post-title}

::: {.archive-year}
2018
:::

-   [11/01]{.archive-post-date} [Android
    系统开发源码环境搭建](/2018/11/01/android-system-develop-0/){.archive-post-title}
-   [10/25]{.archive-post-date}
    [陆奇：除了好代码，工程师怎样才算优秀？](/2018/10/25/How-do-engineers-count-well/){.archive-post-title}
-   [10/01]{.archive-post-date}
    [程序员的修炼-08-阅读之美](/2018/10/01/how-to-stop-sucking-and-be-awesome-instead-8/){.archive-post-title}
-   [09/30]{.archive-post-date}
    [程序员的修炼-07-游戏与编程](/2018/09/30/how-to-stop-sucking-and-be-awesome-instead-7/){.archive-post-title}
-   [09/29]{.archive-post-date}
    [程序员的修炼-06-互联网那些事](/2018/09/29/how-to-stop-sucking-and-be-awesome-instead-6/){.archive-post-title}
-   [09/28]{.archive-post-date}
    [程序员的修炼-05-了解你的用户](/2018/09/28/how-to-stop-sucking-and-be-awesome-instead-5/){.archive-post-title}
-   [09/27]{.archive-post-date}
    [程序员的修炼-04-关于测试的一些思考](/2018/09/27/how-to-stop-sucking-and-be-awesome-instead-4/){.archive-post-title}
-   [09/26]{.archive-post-date} [程序员的修炼-03-Web
    设计原则](/2018/09/26/how-to-stop-sucking-and-be-awesome-instead-3/){.archive-post-title}
-   [09/20]{.archive-post-date}
    [程序员的修炼-02-编程之道](/2018/09/20/how-to-stop-sucking-and-be-awesome-instead-2/){.archive-post-title}
-   [09/19]{.archive-post-date}
    [程序员的修炼-01-绝地反击之术](/2018/09/19/how-to-stop-sucking-and-be-awesome-instead-1/){.archive-post-title}
-   [09/13]{.archive-post-date} [Android
    系统不释放内存吗？](/2018/09/13/android-memory/){.archive-post-title}
-   [08/13]{.archive-post-date} [关于 Android
    系统流畅性的一些思考](/2018/08/13/Some-Thoughts-on-the-Fluency-of-Android/){.archive-post-title}
-   [05/20]{.archive-post-date} [知乎 救救你的
    StartingWindow](/2018/05/20/zhihu-startingwindow/){.archive-post-title}
-   [05/07]{.archive-post-date} [「置顶」Android
    性能优化必知必会](/2018/05/07/Android-performance-optimization-skills-and-tools/){.archive-post-title}
-   [01/06]{.archive-post-date} [2017 年度好物推荐 -
    给辛勤工作的自己一点奖励](/2018/01/06/2017%E5%B9%B4%E5%BA%A6%E6%9C%80%E6%8E%A8%E8%8D%90/){.archive-post-title}

::: {.archive-year}
2017
:::

-   [04/23]{.archive-post-date}
    [2017](/2017/04/23/About-work/){.archive-post-title}

::: {.archive-year}
2016
:::

-   [04/05]{.archive-post-date} [Android Bottom navigation
    规范二：样式、行为与规格](/2016/04/05/android-bottom-bar-2/){.archive-post-title}
-   [04/05]{.archive-post-date} [Android Bottom Navigation
    规范一：使用方法](/2016/04/05/android-bottom-bar-1/){.archive-post-title}

::: {.archive-year}
2015
:::

-   [12/31]{.archive-post-date} [Android 中如何计算 App
    的启动时间？](/2015/12/31/How-to-calculation-android-app-lunch-time/){.archive-post-title}
-   [12/29]{.archive-post-date} [Android 应用启动优化:一种 DelayLoad
    的实现和原理(下篇)](/2015/12/29/Android%E5%BA%94%E7%94%A8%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96-%E4%B8%80%E7%A7%8DDelayLoad%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%92%8C%E5%8E%9F%E7%90%86-%E4%B8%8B%E7%AF%87/){.archive-post-title}
-   [11/18]{.archive-post-date} [Android 应用启动优化 - 一种 DelayLoad
    的实现和原理(上篇)](/2015/11/18/Android-app-lunch-optimize-delay-load/){.archive-post-title}
-   [08/12]{.archive-post-date} [Android hwui 中 RenderThread
    工作流程](/2015/08/12/AndroidL-hwui-RenderThread-workflow/){.archive-post-title}
-   [08/05]{.archive-post-date} [Java7 HashMap
    源码分析](/2015/08/05/HashMap/){.archive-post-title}
-   [07/20]{.archive-post-date} [Android 代码内存优化建议 - OnTrimMemory
    优化](/2015/07/20/Android-Performance-Memory-onTrimMemory/){.archive-post-title}
-   [07/20]{.archive-post-date} [Android 代码内存优化建议 - Android
    资源篇](/2015/07/20/Android-Performance-Memory-AndroidResource/){.archive-post-title}
-   [07/20]{.archive-post-date} [Android 代码内存优化建议 - Android
    官方篇](/2015/07/20/Android-Performance-Memory-Google/){.archive-post-title}
-   [07/20]{.archive-post-date} [Android 代码内存优化建议 - Java
    官方篇](/2015/07/20/Android-Performance-Memory-Java/){.archive-post-title}
-   [05/29]{.archive-post-date} [Nexus6 with Android M
    开启多窗口模式](/2015/05/29/Nexus6-with-Android-M/){.archive-post-title}
-   [05/06]{.archive-post-date} [细说 Java
    单例模式](/2015/05/06/Java-Singleton/){.archive-post-title}
-   [04/19]{.archive-post-date} [Android 性能优化典范 - Profile GPU
    Rendering](/2015/04/19/Android-Performance-Patterns-4/){.archive-post-title}
-   [04/19]{.archive-post-date} [Android 性能优化典范 - Understanding
    VSYNC](/2015/04/19/Android-Performance-Patterns-3/){.archive-post-title}
-   [04/19]{.archive-post-date} [Android 性能优化典范之 Understanding
    Overdraw](/2015/04/19/Android-Performance-Patterns-2/){.archive-post-title}
-   [04/19]{.archive-post-date} [Android 性能优化典范 - Render
    Performance](/2015/04/19/Android-Performance-Patterns-1/){.archive-post-title}
-   [04/19]{.archive-post-date} [Android
    性能优化典范综述](/2015/04/19/Android-Performance-Patterns/){.archive-post-title}
-   [04/11]{.archive-post-date} [Android 内存优化之三 - 打开 MAT 中的
    Bitmap
    原图](/2015/04/11/AndroidMemory-Open-Bitmap-Object-In-MAT/){.archive-post-title}
-   [04/11]{.archive-post-date} [Android 内存优化之二 -
    MAT使用进阶](/2015/04/11/AndroidMemory-Usage-Of-MAT-Pro/){.archive-post-title}
-   [04/11]{.archive-post-date} [Android 内存优化(1) - MAT
    使用入门](/2015/04/11/AndroidMemory-Usage-Of-MAT/){.archive-post-title}
-   [03/31]{.archive-post-date}
    [Android性能优化后续](/2015/03/31/android-performance-case-study-follow-up/){.archive-post-title}
-   [03/15]{.archive-post-date} [Android 小技巧 -
    5](/2015/03/15/android-tips-round-up-5/){.archive-post-title}
-   [03/15]{.archive-post-date} [Android 小技巧 -
    4](/2015/03/15/android-tips-round-up-4/){.archive-post-title}
-   [03/15]{.archive-post-date} [Android 小技巧
    -3](/2015/03/15/android-tips-round-up-3/){.archive-post-title}
-   [02/04]{.archive-post-date} [为 Nexus5 编译 AndroidL
    固件](/2015/02/04/build-rom-for-nexus5/){.archive-post-title}
-   [01/30]{.archive-post-date} [Android 性能优化之 Systrace
    工具介绍(一)](/2015/01/30/android-performance-tools-systrace-1/){.archive-post-title}
-   [01/16]{.archive-post-date} [使用 Android Studio 查看 Android
    Lollipop
    源码](/2015/01/16/view-android-source-code-with-androidstudio/){.archive-post-title}
-   [01/13]{.archive-post-date} [Android 性能优化之过渡绘制 -
    实战篇](/2015/01/13/android-performance-optimization-overdraw-2/){.archive-post-title}

::: {.archive-year}
2014
:::

-   [10/20]{.archive-post-date} [Android 性能优化之过渡绘制 -
    理论篇](/2014/10/20/android-performance-optimization-overdraw-1/){.archive-post-title}
-   [06/03]{.archive-post-date} [Android 小技巧 -
    如何让EditText不自动获取焦点](/2014/06/03/android-edittext-do-not-auto-get-focus/){.archive-post-title}
-   [05/31]{.archive-post-date} [Android 小技巧
    -2](/2014/05/31/android-tips-round-up-2/){.archive-post-title}
-   [05/28]{.archive-post-date} [Android 小技巧 -
    1](/2014/05/28/android-tips-round-up-1/){.archive-post-title}
-   [05/02]{.archive-post-date} [Android Tools -
    Log2File](/2014/05/02/android_log_to_file/){.archive-post-title}
-   [04/01]{.archive-post-date} [Android
    Service：开发自己的通知中心(2):辅助性服务实战](/2014/04/01/android-service-build-your-own-notification-servers-app/){.archive-post-title}
-   [03/25]{.archive-post-date} [Android:Ubuntu 下执行 Adb
    命令找不到设备](/2014/03/25/ubuntu-adb-can-not-find-devices/){.archive-post-title}
-   [03/17]{.archive-post-date} [Android
    Service：开发自己的通知中心(1):辅助性服务介绍](/2014/03/17/android-build-your-own-android-notification-service-app/){.archive-post-title}
:::
:::

::: {.sidebar-panel-tags}
::: {.sidebar-tags-name}
[ []{.iconfont-archer} 年度推荐 ]{.sidebar-tag-name tags="年度推荐"} [
[]{.iconfont-archer} Startup ]{.sidebar-tag-name tags="Startup"} [
[]{.iconfont-archer} Memory ]{.sidebar-tag-name tags="Memory"} [
[]{.iconfont-archer} Android ]{.sidebar-tag-name tags="Android"} [
[]{.iconfont-archer} Jank ]{.sidebar-tag-name tags="Jank"} [
[]{.iconfont-archer} Perfetto ]{.sidebar-tag-name tags="Perfetto"} [
[]{.iconfont-archer} Systrace ]{.sidebar-tag-name tags="Systrace"} [
[]{.iconfont-archer} 年终总结 ]{.sidebar-tag-name tags="年终总结"} [
[]{.iconfont-archer} 知识星球 ]{.sidebar-tag-name tags="知识星球"} [
[]{.iconfont-archer} Performance ]{.sidebar-tag-name
tags="Performance"} [ []{.iconfont-archer} AOSP ]{.sidebar-tag-name
tags="AOSP"} [ []{.iconfont-archer} Framework ]{.sidebar-tag-name
tags="Framework"} [ []{.iconfont-archer} AIW ]{.sidebar-tag-name
tags="AIW"} [ []{.iconfont-archer} Open Source ]{.sidebar-tag-name
tags="Open Source"} [ []{.iconfont-archer} 随笔 ]{.sidebar-tag-name
tags="随笔"} [ []{.iconfont-archer} Agent OS ]{.sidebar-tag-name
tags="Agent OS"} [ []{.iconfont-archer} AI ]{.sidebar-tag-name
tags="AI"} [ []{.iconfont-archer} OpenAI ]{.sidebar-tag-name
tags="OpenAI"} [ []{.iconfont-archer} Mobile OS ]{.sidebar-tag-name
tags="Mobile OS"} [ []{.iconfont-archer} ANR ]{.sidebar-tag-name
tags="ANR"} [ []{.iconfont-archer} RenderThread ]{.sidebar-tag-name
tags="RenderThread"} [ []{.iconfont-archer} Tools ]{.sidebar-tag-name
tags="Tools"} [ []{.iconfont-archer} Linux ]{.sidebar-tag-name
tags="Linux"} [ []{.iconfont-archer} Java ]{.sidebar-tag-name
tags="Java"} [ []{.iconfont-archer} BlastBufferQueue
]{.sidebar-tag-name tags="BlastBufferQueue"} [ []{.iconfont-archer}
性能优化 ]{.sidebar-tag-name tags="性能优化"} [ []{.iconfont-archer}
线程优化 ]{.sidebar-tag-name tags="线程优化"} [ []{.iconfont-archer}
卡顿优化 ]{.sidebar-tag-name tags="卡顿优化"} [ []{.iconfont-archer}
电量优化 ]{.sidebar-tag-name tags="电量优化"} [ []{.iconfont-archer}
ANR分析 ]{.sidebar-tag-name tags="ANR分析"} [ []{.iconfont-archer}
页面响应 ]{.sidebar-tag-name tags="页面响应"} [ []{.iconfont-archer}
Binder ]{.sidebar-tag-name tags="Binder"} [ []{.iconfont-archer} SQL
]{.sidebar-tag-name tags="SQL"} [ []{.iconfont-archer} TraceProcessor
]{.sidebar-tag-name tags="TraceProcessor"} [ []{.iconfont-archer} Trace
]{.sidebar-tag-name tags="Trace"} [ []{.iconfont-archer} DataFlow
]{.sidebar-tag-name tags="DataFlow"} [ []{.iconfont-archer} SDK
]{.sidebar-tag-name tags="SDK"} [ []{.iconfont-archer} TraceEvent
]{.sidebar-tag-name tags="TraceEvent"} [ []{.iconfont-archer} heapprofd
]{.sidebar-tag-name tags="heapprofd"} [ []{.iconfont-archer} Field
]{.sidebar-tag-name tags="Field"} [ []{.iconfont-archer} GPU
]{.sidebar-tag-name tags="GPU"} [ []{.iconfont-archer} Power
]{.sidebar-tag-name tags="Power"} [ []{.iconfont-archer} MAT
]{.sidebar-tag-name tags="MAT"} [ []{.iconfont-archer} Camera
]{.sidebar-tag-name tags="Camera"} [ []{.iconfont-archer} Audio
]{.sidebar-tag-name tags="Audio"} [ []{.iconfont-archer} Input
]{.sidebar-tag-name tags="Input"} [ []{.iconfont-archer} Latency
]{.sidebar-tag-name tags="Latency"} [ []{.iconfont-archer} DelayLoad
]{.sidebar-tag-name tags="DelayLoad"} [ []{.iconfont-archer} HashMap
]{.sidebar-tag-name tags="HashMap"} [ []{.iconfont-archer} 程序员
]{.sidebar-tag-name tags="程序员"} [ []{.iconfont-archer} Nexus
]{.sidebar-tag-name tags="Nexus"} [ []{.iconfont-archer} Singleton
]{.sidebar-tag-name tags="Singleton"} [ []{.iconfont-archer} OpenClaw
]{.sidebar-tag-name tags="OpenClaw"} [ []{.iconfont-archer} AI Agent
]{.sidebar-tag-name tags="AI Agent"} [ []{.iconfont-archer} 效率工具
]{.sidebar-tag-name tags="效率工具"} [ []{.iconfont-archer} 本地模型
]{.sidebar-tag-name tags="本地模型"} [ []{.iconfont-archer}
SmartPerfetto ]{.sidebar-tag-name tags="SmartPerfetto"} [
[]{.iconfont-archer} MCP ]{.sidebar-tag-name tags="MCP"} [
[]{.iconfont-archer} Android Performance ]{.sidebar-tag-name
tags="Android Performance"} [ []{.iconfont-archer} Gracker
]{.sidebar-tag-name tags="Gracker"} [ []{.iconfont-archer} 设计规范
]{.sidebar-tag-name tags="设计规范"} [ []{.iconfont-archer} Compose
]{.sidebar-tag-name tags="Compose"} [ []{.iconfont-archer} Kotlin
]{.sidebar-tag-name tags="Kotlin"} [ []{.iconfont-archer} Process
]{.sidebar-tag-name tags="Process"} [ []{.iconfont-archer} 卡顿
]{.sidebar-tag-name tags="卡顿"} [ []{.iconfont-archer} 读书笔记
]{.sidebar-tag-name tags="读书笔记"} [ []{.iconfont-archer}
程序员的修炼 ]{.sidebar-tag-name tags="程序员的修炼"} [
[]{.iconfont-archer} 设计原则 ]{.sidebar-tag-name tags="设计原则"} [
[]{.iconfont-archer} 测试 ]{.sidebar-tag-name tags="测试"} [
[]{.iconfont-archer} 了解你的用户 ]{.sidebar-tag-name
tags="了解你的用户"} [ []{.iconfont-archer} 互联网那些事
]{.sidebar-tag-name tags="互联网那些事"} [ []{.iconfont-archer}
游戏与编程 ]{.sidebar-tag-name tags="游戏与编程"} [
[]{.iconfont-archer} 阅读之美 ]{.sidebar-tag-name tags="阅读之美"} [
[]{.iconfont-archer} 利器 ]{.sidebar-tag-name tags="利器"} [
[]{.iconfont-archer} iOS ]{.sidebar-tag-name tags="iOS"} [
[]{.iconfont-archer} Design ]{.sidebar-tag-name tags="Design"} [
[]{.iconfont-archer} Smooth ]{.sidebar-tag-name tags="Smooth"} [
[]{.iconfont-archer} 体验优化 ]{.sidebar-tag-name tags="体验优化"} [
[]{.iconfont-archer} 用户体验 ]{.sidebar-tag-name tags="用户体验"}
:::

::: {.iconfont-archer .sidebar-tags-empty}

:::

::: {.tag-load-fail style="display: none; color: #ccc; font-size: 0.6rem;"}
缺失模块，请参考主题文档进行安装配置：https://github.com/fi3ework/hexo-theme-archer\#%E5%AE%89%E8%A3%85%E4%B8%BB%E9%A2%98
:::

::: {.sidebar-tags-list}
:::
:::

::: {.sidebar-panel-categories}
::: {.sidebar-categories-name}
[ []{.iconfont-archer} 随笔 ]{.sidebar-category-name categories="随笔"}
[ []{.iconfont-archer} 读书笔记 ]{.sidebar-category-name
categories="读书笔记"} [ []{.iconfont-archer} Android
]{.sidebar-category-name categories="Android"} [ []{.iconfont-archer}
AI ]{.sidebar-category-name categories="AI"} [ []{.iconfont-archer}
Java ]{.sidebar-category-name categories="Java"} [ []{.iconfont-archer}
Android性能优化 ]{.sidebar-category-name categories="Android性能优化"} [
[]{.iconfont-archer} SmartPerfetto ]{.sidebar-category-name
categories="SmartPerfetto"} [ []{.iconfont-archer} 个人页
]{.sidebar-category-name categories="个人页"}
:::

::: {.iconfont-archer .sidebar-categories-empty}

:::

::: {.sidebar-categories-list}
:::
:::
:::
:::

::: {.site-search .site-search-loading .local-search-shell}
::: {.algolia-popup .popup .local-search-popup role="dialog" aria-modal="true" aria-label="Search posts"}
::: {.algolia-search .local-search-header}
::: {.algolia-search-input-icon}
**
:::

::: {.algolia-search-input .local-search-input}
:::

**
:::

::: {.algolia-results .local-search-results}
::: {#local-search-config path="/search.xml" per-page="10" empty="没有找到与「${query}」相关的文章" stats="找到 ${hits} 个结果" data-loading="正在加载搜索索引..." error="搜索索引加载失败，请稍后重试"}
:::

::: {#local-search-stats .algolia-stats}
:::

::: {#local-search-hits}
:::
:::
:::
:::
