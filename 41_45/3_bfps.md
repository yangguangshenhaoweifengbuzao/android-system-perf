核心组件：

  ┌─────────────────┬──────────────────────────────────────────────┐
  │      类名       │                     作用                     │
  ├─────────────────┼──────────────────────────────────────────────┤
  │ BFpsTracker     │ 单个 Layer 的 FPS 追踪器，统计帧数、计算 FPS │
  ├─────────────────┼────────
  │ BFpsTrackerLine │ 管理所有 Layer 的 Tracker，处理事件分发      │
  ├─────────────────┼──────────────────────────────────────────────┤
  │ LayerConfig     │ 定义不同 Layer 类型支持的事件类型            │
  └─────────────────┴──────────────────────────────────────────────┘

  Layer 类型分类：

 enum LayerType {
      UI_EVENT = 0,   // 普通 UI（支持 onfling、动画事件）
      UI_AUTO = 1,    // 自动播放场景（3D、视频、开机动画）
      UI_VIDEO = 2,   // 视频/相机
  };

  帧计数方式：

  // BufferStateLayer::decrementPendingBufferCount()
  mBFpsTracker->incrementPostBufferCount();  // 每次 buffer 提交计数

// Layer::setBounds()
  mScreenBounds 变化时也计数（动画场景）

  FPS 计算逻辑：

  mCurrentFps = postBufferCount * 1000.0 / mGapTime;
  // postBufferCount: 帧数
  // mGapTime: 时间间隔 (ms)

  与 SurfaceFlinger 的集成：

  - 创建 Layer 时注册 Tracker
  - 销毁 Layer 时移除 Tracker
  - 通过 Binder 接收 Framework 层事件（code = 10008)

 新增接口：

  public interface IUxEventStub {
      void onEventStart(int pid, String windowName, String eventName, double preferFps, int duration);
      void onEventStop(int pid, String windowName, String eventName);
      void sendSceneFps(int pid, String windowName, String sceneName, double currentFps, double preferFps, int duration);
  }

  埋点位置：

  ┌────────────────────────────────┬──────────────┬────────────────────────────┐
  │              位置              │     场景     │           事件名           │
  ├────────────────────────────────┼──────────────┼────────────────────────────
   │ ViewRootImpl.deliverInputEvent │ 触摸事件     │ input_down/move/up         │
  ├────────────────────────────────┼──────────────┼────────────────────────────┤
  │ DisplayPolicy.InterceptMotion  │ Fling 手势   │ onfling (duration > 200ms) │
  ├────────────────────────────────┼──────────────┼──────────
   │ RemoteAnimationController      │ 远程动画     │ anim-remote                │
  ├────────────────────────────────┼──────────────┼────────────────────────────┤
  │ ScreenRotationAnimation        │ 屏幕旋转     │ anim-rotation (600ms 超时) │
  ├────────────────────────────────┼──────────────┼──────────────
  │ SurfaceAnimator                │ App 过渡动画 │ anim-trans                 │
  └────────────────────────────────┴──────────────┴────────────────────────────┘

  通信方式：

  // 通过 Binder oneway 发送到 SurfaceFlinger
  getServer().transact(SURFACEFLINGER_UX_BFPS_CODE, data, null, FLAG_ONEWAY);
