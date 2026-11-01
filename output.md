Android Perfetto 深入理解 Vsync 机制与性能分析
======================================================

## 什么是-Vsync
---------------------------------------------------------

VSync（Vertical Synchronization，垂直同步）给帧生产和显示合成提供共同的时间参考。App、SurfaceFlinger
与显示硬件会按各自的调度相位工作，不能把它们画成同一个瞬间。

在没有 Vsync 机制之前，常见问题是屏幕撕裂（Screen
Tearing）。当显示器读取 framebuffer 的同时，GPU
写入了下一帧，就会在同一次刷新中出现上下两部分不一致的画面。

###  Vsync-解决什么问题？

Android 用 VSync 安排帧回调和合成时点，再用 BufferQueue 与 fence 协调读写：

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


## Android-中-Vsync-的基本工作原理
-----------------------------------------------------------------------------------------------------------------
Android 系统的 Vsync
实现比基本概念复杂得多，需要考虑多个不同的渲染组件，以及它们之间的协调工作。

### Vsync-信号的分层架构

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


## 在-Perfetto-中观察-Vsync
---------------------------------------------------------------------------------------------

Perfetto trace 中包含多个与 Vsync 相关的 Track，理解这些 Track
的含义有助于分析性能问题。

**在 SurfaceFlinger 进程中**：

1.  **vsync-app**\
    这份 Trace 中该 counter 在 0 和 1
    之间变化，可用变化时点辅助观察调度节奏。它不是某个指定 App
    已收到回调的证明。\
    <img width="2102" height="156" alt="image" src="https://github.com/user-attachments/assets/2a3f1d6b-6887-48bf-9616-91b31fd62531" />
2.  **vsync-sf**\
    显示 SurfaceFlinger Vsync 信号状态。无 Vsync Offset 时与 `vsync-app`
    同步变化。\
    <img width="2022" height="258" alt="image" src="https://github.com/user-attachments/assets/e185a7a9-911e-400e-b6a6-0d4724aca2c3" />
3.  **vsync-appSf**\
    Android 13+ 新增，服务于需要与 SurfaceFlinger 同步的特殊
    Choreographer 客户端。\
    <img width="2200" height="280" alt="image" src="https://github.com/user-attachments/assets/a5b598d7-43e3-449e-992d-eab51f4dbe20" />
4.  **HW\_VSYNC**\
    这条 counter 记录硬件 VSync 的启用状态，值为 1/0
    时分别表示开启/关闭。不要把每次状态变化当成一次 App 帧回调。\
    <img width="1832" height="332" alt="image" src="https://github.com/user-attachments/assets/35fcfeda-0778-4330-8e44-19630b150e57" />

**在应用进程中**：

`FrameDisplayEventReceiver.onVsync` Slice Track：\
显示应用接收 Vsync 信号的时间点。该事件连接通过 Binder 建链、通过
`BitTube`/Looper 通道分发事件，时间可能略晚于 SurfaceFlinger 中的
`vsync-app`。
<img width="2630" height="504" alt="image" src="https://github.com/user-attachments/assets/73048b1a-9dd6-4c17-a602-43aef9b1b1f8" />

`UI Thread` Slice Track：\
包含 `Choreographer#doFrame` 及相关的 Input、Animation、Traversal 等
Slice。每个 `doFrame` 对应一帧的处理工作。
<img width="2660" height="552" alt="image" src="https://github.com/user-attachments/assets/1283dc19-d528-42c3-bce9-8f51e070bdbd" />

`RenderThread` Slice Track：\
包含 `DrawFrame`、`syncAndDrawFrame`、`queueBuffer` 等
Slice，对应渲染线程工作。
<img width="2262" height="356" alt="image" src="https://github.com/user-attachments/assets/2d8cbe55-aa00-4dd9-90c2-6888126aeb08" />

## Android-App-每一帧是如何基于-Vsync-工作的
------------------------------------------------------------------------------------------------------------------------------------------------

下图把 App 请求回调、主线程执行和 SurfaceFlinger
合成放在一条时间线上。不同帧可能跳过某些阶段，图中箭头表示调查顺序，不表示所有工作都在一个
VSync 周期内完成。
<img width="2508" height="2002" alt="image" src="https://github.com/user-attachments/assets/18c6876d-5a69-4560-a74e-6f79484459fc" />

### 流程总览（按顺序）

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

### App-什么时候会申请-Vsync-信号

应用并不是时刻都在申请 Vsync 信号的。Vsync
信号是按需申请的，只有在以下情况下，应用才会向系统申请下一个 Vsync：

**常见的触发申请 Vsync 的场景**：

1.  **UI 更新需求**：当 View 调用 `invalidate()` 时
2.  **动画执行**：ValueAnimator、ObjectAnimator 等动画开始时
3.  **用户交互**：触摸事件、按键事件等需要 UI 响应时
4.  **数据变化**：RecyclerView 数据更新、TextView 文本改变等

### App-申请-Vsync-的完整流程

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

### 主线程如何监听-Vsync-信号

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

