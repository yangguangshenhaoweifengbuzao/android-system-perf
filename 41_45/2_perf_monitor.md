### 1.概述
PerfMonitorService 是一个 Android 系统级性能监控服务，用于实时监控和管理系统性能数据。该服务主要运行在 Android 框架层，通过 Binder IPC 提供跨进程通信能力，能够监控 CPU、GPU、磁盘等硬件资源的使用情况，并在异常发生时自动收集和上传相关诊断信息。
### 2.1目录
```
PerfMonitorService/
├── Android.bp                 # 编译配置文件
├── README.md                  # 项目说明
├── config.pbtxt               # Perfetto 默认配置
├── config_detail.pbtxt        # Perfetto 详细配置
├── config_simple.pbtxt        # Perfetto 简化配置
├── src/                       # 源代码目录
│   ├── main.cpp               # 服务入口
│   ├── PerfMonitorService.h/cpp    # 核心服务类
│   ├── AbstractMonitor.h      # 监控器抽象接口
│   ├── CpuMonitor.h/cpp       # CPU 监控器
│   ├── GpuMonitor.h/cpp       # GPU 监控器
│   ├── DiskMonitor.h/cpp      # 磁盘监控器
│   ├── ConfigManager.h/cpp    # 配置管理
│   ├── RemoteStrategy.h/cpp   # 云端策略管理
│   ├── PerfettoManager.h/cpp  # Perfetto 追踪管理
│   ├── FrameInfoManager.h/cpp # 帧信息管理
│   ├── AprUpload.h/cpp        # 文件上传管理
│   ├── CollectInfo.h/cpp      # 信息收集器
│   ├── AccStatManager.h/cpp   # ACC 状态管理
│   ├── SysDataReportServer.h/cpp # Binder 服务端
│   ├── dataBase/              # 数据库模块
│   │   └ DataBaseManager.h/cpp  # SQLite 数据库管理
│   ├── lib/                   # 库模块
│   │   ├── PerfMonitorThread.h/cpp # 帧监控线程
│   ├── Utils.h/cpp            # 工具函数
│   ├── RingBuffer.h           # 环形缓冲区
│   └── ShellCommand.h         # Shell 命令封装
└── test/                      # 测试代码目录
```
### 2.2 架构图

```
                    ┌─────────────────────────────────────────┐
                    │           PerfMonitorService            │
                    │         (核心服务调度中心)               │
                    └────────────────┬────────────────────────┘
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  Monitor 模块 │          │  收集/上传模块   │          │   配置管理模块   │
├───────────────┤          ├─────────────────┤          ├─────────────────┤
│ - CpuMonitor  │          │ - CollectInfo   │          │ - ConfigManager │
│ - GpuMonitor  │          │ - AprUpload     │          │ - RemoteStrategy│
│ - DiskMonitor │          │ - PerfettoMgr   │          │                 │
│               │          │ - FrameInfoMgr  │          │                 │
└───────────────┘          └─────────────────┘          └─────────────────┘
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     SysDataReportServer (Binder IPC)                  │
│                    (对外数据上报服务接口)                               │
└────────────
                                     │
                                     ▼
                    ┌─────────────────────────────────────────┐
                    │         DataBaseManager (SQLite)        │
                    │            (数据持久化)                  │
                    └────────
```
### 3.核心模块详解
### 3.1 PerfMonitorService (核心服务)

**文件**: `src/PerfMonitorService.h/cpp`

**功能**:
- 系统性能监控的主控制中心
- 管理所有监控器 (Monitor) 的注册、初始化和生命周期
- 基于 Looper 实现定时调度监控任务
- 支持远程策略动态配置

### 3.2 AbstractMonitor (监控器抽象接口)

**文件**: `src/AbstractMonitor.h`

**功能**:
- 定义所有监控器的统一接口规范
- 提供周期性监控 (`mPeriod`) 和基础生命周期管理

### 3.3 CpuMonitor (CPU 监控器)

**文件**: `src/CpuMonitor.h/cpp`

**功能**:
- 实时监控 CPU 负载
- 通过读取 `/proc/stat` 计算系统负载
- 异常状态检测与分级 (NORMAL/ABNORMAL/PERSISTENT_ABNORMAL)
- 支持云端阈值配置

**异常判定逻辑**:
- 连续超过阈值次数达到 `mCpuAbnormalCountThreshold` 时判定为异常
- 使用环形缓冲区 (`RingBuffer`) 存储历史数据用于趋势分析

### 3.4 GpuMonitor (GPU 监控器)

**文件**: `src/GpuMonitor.h/cpp`

**功能**:
- 监控 GPU 负载和频率 (针对 Qualcomm 平台)
- 读取 GPU busy percentage 和当前频率
- 支持多级频率档位管理

### 3.5 DiskMonitor (磁盘监控器)

**文件**: `src/DiskMonitor.h/cpp`

**功能**:
管理磁盘存储空间，确保性能追踪文件不会耗尽系统存储，通过时间、目录大小、分区使用率三种策略自动清理过期文件。

### 3.6 ConfigManager (配置管理器)

**文件**: `src/ConfigManager.h/cpp`

**功能**:
- 管理所有配置项的默认值、属性键、云端键
- 提供统一的配置获取接口
- 支持多种数据类型 (BOOL/INT/FLOAT/STRING)

### 3.7 RemoteStrategy (云端策略管理)

**文件**: `src/RemoteStrategy.h/cpp`

**功能**:
- 接收云端策略下发并通知本地观察者
- 实现配置优先级: **本地属性 > 云端配置 > 代码默认值**
- 基于 `StrategyManager` 和 Binder IPC 实现

### 3.8 PerfettoManager (Perfetto 追踪管理)

**文件**: `src/PerfettoManager.h/cpp`

**功能**:
- 管理 Perfetto 系统追踪工具的启停
- 支持多种场景 (后台/UI/CMD)
- 支持多级配置 (simple/default/detail)
- ACC 状态联动 (熄火时自动停止追踪)

### 3.9 FrameInfoManager (帧信息管理)

**文件**: `src/FrameInfoManager.h/cpp`

**功能**:
- 管理各进程的帧率信息
- 基于 `LruCache` 按 PID 存储帧数据
- 支持帧率异常触发上报

### 3.10 CollectInfo (信息收集器)

**文件**: `src/CollectInfo.h/cpp`

**功能**:
- 异常事件发生时的信息收集中心
- 支持多种事件类型收集 (ANR、CPU 超限、启动超时等)
- 执行文件复制、重命名、压缩等操作

### 3.11 AprUpload (文件上传管理)

**文件**: `src/AprUpload.h/cpp` (`FileUploadManager` 类)

**功能**:
- 管理上传文件记录和黑名单
- 控制上传频率和次数限制
- 支持上传计数管理 (日/月限制)
支持的异常类型:

  ┌───────────────────┬──────────────┬──────────────────────────────────────────────┐
  │       类型        │     说明      │                 对应 DTC 码                  │
  ├───────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ CpuOverThreshhold │ CPU 超限     │ DTC_PERFORMANCE_ABNORMAL_CPU_OVER_THRESHHOLD │
  ├───────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ Key-ANR           │ 关键 ANR     │ DTC_PERFORMANCE_ABNORMAL_KEY_ANR             │
  ├───────────────────┼──────────────┼────────────────────────────────────────
  │ AppLaunchTimeout  │ 启动超时     │ DTC_PERFORMANCE_ABNORMAL_LAUNCH_TIMEOUT      │
  
### 3.12 AccStatManager (ACC 状态管理)

**文件**: `src/AccStatManager.h/cpp`

**功能**:
- 监控车辆 ACC (电源) 状态变化
- 通过 `BYDAutoManager` 获取车控信号
- 状态变更时通知观察者

### 3.13 SysDataReportServer (Binder 服务)（在其他进程里面埋点，通过binder传递异常信息，如anr实在system_server里面买点）

**文件**: `src/SysDataReportServer.h/cpp`

**功能**:
- 提供跨进程数据上报接口
- 基于 AIDL 定义 Binder 服务
- 处理各类型异常事件上报

### 3.14 DataBaseManager (数据库管理)

**文件**: `src/dataBase/DataBaseManager.h/cpp`

**功能**:
- SQLite 数据库封装
- 异步写入队列处理
- 支持设备信息、上传记录等数据持久化
表结构说明

  ┌─────────────────────┬──────────────────────────────────────────────────┬────────────────────────────┐  
  │        表名         │                       字段                        │            用途            │  
  ├─────────────────────┼──────────────────────────────────────────────────┼───
  │ alreadyUploadedFile │ id, fileNAME(主键), uploadTime                   │ 记录已上传的文件名和时间     │  
  ├─────────────────────┼──────────────────────────────────────────────────┼────────────────────────────┤  
  │ deviceInfo          │ iccid, vin, inswVer, outswVer, mcuVer, carModel, │ 存储设备/车辆/软件版本信息   │  
  │                     │  carBrand, platform, uiVer, updateTime           │                            │  
  ├─────────────────────┼──────────────────────────────────────────────────┼────────────────────────────┤  
  │ test                │ id, name, age                                    │ 测试用途
  WAL 模式优化

  // 开启 Write-Ahead Logging
  sqlite3_exec(mDatabase, "PRAGMA journal_mode=WAL;");
  sqlite3_exec(mDatabase, "PRAGMA wal_autocheckpoint=500;");
### 3.15 RingBuffer (环形缓冲区)

**文件**: `src/RingBuffer.h`

**功能**:
- 固定大小的环形缓冲区模板类
- 用于存储历史监控数据,自动丢弃旧数据
- 支持滑动窗口数据分析

## 4. 服务启动流程

**入口文件**: `src/main.cpp`

```cpp
int main() {
    // 1. 等待启动条件检查 (远程策略 + 本地属性)
    while (!PerfMonitorService::isStart()) {
        sleep(REMOTESTRATEGY_CHECK_PERIODS[checkCount]);
    }

    // 2. 注册 Binder 服务
    sm->addService(SysDataReportServer::getServiceName(), new SysDataReportServer());

    // 3. 初始化核心组件
    AccStatManager::getInstance();
    RemoteStrategy::getInstance();
    Config::getInstance();

    // 4. 创建 Looper 线程池
    mainLooper = Looper::prepare(true);
    secondLooper = new Looper(true);

    // 5. 初始化上传/删除管理器
    FileUploadManager fum(secondLooper);
    FileDeleteManager fdm(fum, DELETE_PERIOD, secondLooper);

    // 6. 初始化信息收集器
    CollectInfo::getInstance()->start(secondLooper);

    // 7. 启动 Perfetto 追踪
    PerfettoManager::getInstance()->start(secondLooper);

    // 8. 注册监控器
    PerfMonitorService pms(mainLooper);
    pms.registerMonitor(CPU_LOAD_MONITOR, cpuMonitor);
    pms.registerMonitor(GPU_MONITOR, gpuMonitor);

    // 9. 进入主循环
    while (PerfMonitorSwitch == 1) {
        mainLooper->pollOnce(-1);
    }
}
```
## 5. Perfetto 配置说明

Perfetto 是 Android 系统级追踪工具，本服务提供三套配置:

| 配置文件 | 用途 | Buffer 大小 |
|---------|------|------------|
| `config_simple.pbtxt` | 简化追踪 | 较小 |
| `config.pbtxt` | 默认追踪 | 102400 KB |
| `config_detail.pbtxt` | 详细追踪 | 最大 |

## 6.Input Latency Monitor
### 延迟分段计算
在inputdispatcher里面埋点记录各个过程时间

延迟段	计算方式	含义

allDelay	finishTime - eventTime	总延迟 (事件产生到完成)

deliveryDelay	deliveryTime - eventTime	分发延迟 (到开始分发)

delivery2EnOqDelay	enOqTime - deliveryTime	分发到入队延迟

enOq2EnWqDelay	enWqTime - enOqTime	OutboundQueue 到 WaitQueue 延迟

deOq2ConsumeDelay	consumeTime - enWqTime	WaitQueue 到消费延迟

## 7.帧率监控和app启动监控  libs/hwui/JankTracker.cpp     /     ActivityMetricsLogger.java
### 7.1 app启动

cold	1	冷启动 - 进程不存在，需创建新进程

warm	2	温启动 - 进程存在但 Activity 需重建

hot	3	热启动 - 进程和 Activity 都存在，只需恢复

// 1. startActivity 时记录开始时间

void setStartTime(int dataType, long uptimes) {
    mTimes[dataType] = uptimes;
    // dataType = AMS_START_ACTIVITY
}

// 2. bindApplication 时记录绑定时间

// dataType = AMS_BIND_APPLICATION

// 3. 首帧显示时记录显示时间

// dataType = AMS_DISPLAYED

<com.android.settings/.Settings (10123,12345678,1.2.3) 
 startActivityTime=1234567890 bindApplicationTime=1234567900 
 displayedTime=1234568100 delay=210ms type=cold 
 dateTime=2025-04-12 10:30:45>
 
### 8.2 帧率监控
int64_t realTotalDuration = frame.duration(FrameInfoIndex::Vsync,
        FrameInfoIndex::FrameCompleted);

#### 8.2.1 FrameInfoIndex 完整枚举定义

```cpp
enum FrameInfoIndex {
    // ========== 信号与意图 ==========
    Flags,                      // 帧标志位
    FrameTimelineVsyncId,       // FrameTimeline Vsync ID
    Vsync,                      // Vsync 时间戳 (帧开始)
    ExpectedVsync,              // 期望的 Vsync 时间
    ExpectedEndTime,            // 期望结束时间
    
    // ========== 输入处理阶段 ==========
    InputHandlingStart,         // 输入处理开始
    InputHandling,              // 输入处理结束
    
    // ========== 动画阶段 ==========
    AnimationCallbackStart,     // 动画回调开始
    AnimationCallback,          // 动画回调结束
    
    // ========== 遍历阶段 ==========
    PerformTraversalsStart,     // performTraversals() 开始
    PerformTraversals,          // performTraversals() 结束
    
    // ========== 绘制阶段 ==========
    DrawStart,                  // draw() 开始
    Draw,                       // draw() 结束 (录制 DisplayList)
    
    // ========== 同步阶段 ==========
    SyncStart,                  // sync() 开始
    Sync,                       // sync() 结束 (资源同步)
    
    // ========== GPU 阶段 ==========
    IssueDrawCommandsStart,     // GPU 命令下发开始
    IssueDrawCommands,          // GPU 命令下发结束
    // ========== 缓冲区阶段 ==========
    SwapBuffers,                // eglSwapBuffers() 调用
    FrameCompleted,             // 帧完成时间戳
    
    // ========== 额外信息 ==========
    DequeueBuffer,              // dequeueBuffer 耗时
    QueueBuffer,                // queueBuffer 耗时
};
```
#### 8.2.2 典型卡顿场景分析

##### 场景 1: 列表滚动卡顿

```
帧耗时分布 (总耗时 45ms):
┌─────────────────────────────────────────────────────────────┐
│ Input:    ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ Animation:██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5ms         │
│ Traversal:████████████████░░░░░░░░░░░░░░░░░░  18ms ⚠️     │
│ Draw:     ████████░░░░░░░░░░░░░░░░░░░░░░░░░░  6ms         │
│ Sync:     ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ GPU:      ████████░░░░░░░░░░░░░░░░░░░░░░░░░░  6ms         │
│ Swap:     ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  4ms         │
└─────────────────────────────────────────────────────────────┘

问题定位: Traversal 阶段耗时过长 (18ms)
原因分析: RecyclerView Item 布局复杂，onMeasure 中有耗时计算
优化建议: 
  1. 简化 Item 布局层级
  2. 使用 DiffUtil 替代 notifyDataSetChanged
  3. 预计算 Item 高度避免重新测量
```

##### 场景 2: 动画卡顿

```
帧耗时分布 (总耗时 32ms):
┌─────────────────────────────────────────────────────────────┐
│ Input:    ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1ms         │
│ Animation:████████████████████████░░░░░░░░░░  20ms ⚠️     │
│ Traversal:████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ Draw:     ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ Sync:     ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1ms         │
│ GPU:      ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  4ms         │
└─────────────────────────────────────────────────────────────┘

问题定位: Animation 阶段耗时过长 (20ms)
原因分析: 多个属性动画同时运行，动画监听器中有复杂计算
优化建议:
  1. 使用 RenderThread 动画
  2. 减少同时运行的动画数量
  3. 将复杂计算移出动画回调
```

##### 场景 3: 图片加载卡顿

```
帧耗时分布 (总耗时 38ms):
┌─────────────────────────────────────────────────────────────┐
│ Input:    ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1ms         │
│ Animation:██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1ms         │
│ Traversal:████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ Draw:     ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  3ms         │
│ Sync:     ███████████████████████████░░░░░░░  25ms ⚠️     │
│ GPU:      ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5ms         │
└─────────────────────────────────────────────────────────────┘

问题定位: Sync 阶段耗时过长 (25ms)
原因分析: 大图解码和 GPU 上传阻塞优化建议:
  1. 使用异步图片加载库 (Glide/Coil)
  2. 预加载图片到内存
  3. 使用 hardware Bitmap
  4. 降低图片分辨率
```

#### 8.2.3 帧耗时计算代码

```cpp
// JankTracker.cpp
struct FrameTiming {
    int64_t inputTime;      // InputHandling - Vsync
    int64_t animationTime;  // AnimationCallback - InputHandling
    int64_t traversalTime;  // PerformTraversalsStart - AnimationCallback
    int64_t drawTime;       // DrawStart - PerformTraversalsStart
    int64_t syncTime;       // SyncStart - DrawStart
    int64_t gpuTime;        // IssueDrawCommandsStart - SyncStart
    int64_t swapTime;       // SwapBuffers - IssueDrawCommandsStart
    int64_t totalTime;      // FrameCompleted - Vsync
};

FrameTiming getFrameTiming(FrameInfo& frame) {
    FrameTiming timing;
    timing.inputTime = frame.duration(Vsync, InputHandling);
    timing.animationTime = frame.duration(InputHandling, AnimationCallback);
    timing.traversalTime = frame.duration(AnimationCallback, PerformTraversalsStart);
    timing.drawTime = frame.duration(PerformTraversalsStart, DrawStart);
    timing.syncTime = frame.duration(DrawStart, SyncStart);
    timing.gpuTime = frame.duration(SyncStart, IssueDrawCommandsStart);
    timing.swapTime = frame.duration(IssueDrawCommandsStart, SwapBuffers);
    timing.totalTime = frame.duration(Vsync, FrameCompleted);
    return timing;
}
```
#### 8.2.4 各阶段优化总结

| 阶段 | 正常耗时 | 常见慢因 | 优化方向 |
|-----|---------|---------|---------|
| **Input** | < 2ms | 事件堆积、复杂手势处理 | 异步处理、避免阻塞 |
| **Animation** | < 2ms | 动画过多、监听器耗时 | 使用 RenderThread 动画 |
| **Traversal** | < 4ms | View 层级深、复杂布局 | 扁平化布局、异步 inflate |
| **Draw** | < 4ms | 复杂绘制、对象创建 | 复用对象、硬件加速 |
| **Sync** | < 2ms | 图片解码、资源上传 | 预加载、异步解码 |
| **GPU** | < 4ms | Shader 复杂、过度绘制 | 简化效果、减少层数 |
| **Swap** | < 2ms | GPU 未完成、缓冲区分配 | 增加缓冲区、优化 GPU |

#### 8.2.5 perfetto对应
安卓版本需要支持FrameTimeline 数据源映射才行

android.surfaceflinger.frametimeline 数据源提供完整的帧时间线追踪

#### 8.2.6 上传数据
包含 aplog ， 上报帧记录（按pid排序）， perfetto，只有严重卡顿的帧才会上报，将这些文件打包，通过系统属性/云端 控制上报频率等


## 9. 适用场景

1. **车载 Android 系统** 
2. **性能异常诊断** - CPU/GPU 超限、ANR、启动超时
3. **帧率监控** - UI 卡顿检测与上报
4. **远程策略控制** - 云端下发监控阈值
5. **Perfetto 追踪** - 系统级性能追踪与分析

---

## 10. 总结

PerfMonitorService 是一个完整的系统性能监控解决方案，具备:

- **模块化监控器架构** - CPU/GPU/Disk 等可插拔监控组件
- **Binder IPC  服务** - 跨进程数据上报能力
- **云端策略联动** - 远程配置动态调整
- **Perfetto 集成** - 专业系统追踪工具支持
- **数据持久化** - SQLite 存储设备与上传记录
- **异常自动收集** - 多类型事件触发信息收集与上传

该服务适用于车载 Android 平台的性能监控与诊断场景，能够有效帮助开发人员定位和分析系统性能问题。
