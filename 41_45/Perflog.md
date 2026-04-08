# BPerfLog 性能监控系统架构分析

## 一、系统概述

BPerfLog 是一个 Android 性能监控系统，用于追踪任务执行过程中的各种延迟信息，帮助定位性能瓶颈。系统采用分层架构，从内核驱动采集数据，通过用户空间库传递到 Framework 层。

---
## 三、各层模块详细分析

### 3.1 内核层 (Kernel Layer)

#### 3.1.1 模块结构

内核驱动 `bperf_taskacct.ko` 采用**资源抽象模式**，将不同功能模块化为独立的 "resource"：

| Resource | 文件 | 职责 |
|----------|------|------|
| `res_trace` | `res_trace.c` | Tracepoint 管理与查找 |
| `bperf_task` | `bperf_task.c` | 任务生命周期管理 (radix tree) |
| `task_acct` | `task_acct.c` | 性能数据采集与 ioctl 接口 |

#### 3.1.2 res_trace - Tracepoint 管理

**核心机制：**
- 维护全局 tracepoint hash 表，按首字母分桶 ('a'-'z')
- 提供 `hook_tracepoint()` / `unhook_tracepoint()` API 给其他模块使用

**关键数据结构：**
```c
struct trace_node {
    struct tracepoint *tp;          // 内核 tracepoint 指针
    struct hlist_node node;         // hash 表节点
    struct hlist_node hlist;        // 全链表节点
    const char *trace_name;         // tracepoint 名称
};
```

**监控的 Tracepoint 列表 (res_trace.h)：**
```c
ADDTRACE(sched_process_fork)      // 进程创建
ADDTRACE(sched_process_exit)      // 进程退出
ADDTRACE(sched_switch)            // 调度切换
ADDTRACE(sched_wakeup)            // 进程唤醒
ADDTRACE(sched_stat_sleep)        // 睡眠统计
ADDTRACE(sched_stat_blocked)      // 阻塞统计
ADDTRACE(mm_vmscan_direct_reclaim_begin/end)  // 内存回收
ADDTRACE(android_vh_alloc_pages_slowpath)     // 页分配慢路径
// ... 等
```

#### 3.1.3 bperf_task - 任务管理

**核心机制：**
- 使用 **radix tree** 以 PID 为 key 存储每个任务的监控数据
- 利用 `task_struct->android_oem_data1[GKI_IDX]` 直接指向监控节点，加速访问
- 通过 tracepoint hook 监控任务生命周期和数据采集

#### 3.1.4 task_acct - 数据采集接口

**核心机制：**
- 创建 `/proc/bperf/task_acct` 文件接口
- 提供 ioctl 命令：`ACCT_IOC_START` (开始监控) / `ACCT_IOC_READ` (读取数据)

**ioctl 数据结构 (与用户空间共享)：**
```c
struct acct_info {
    pid_t tid;
    long blkio_delay;              // sync block io 延迟
    long swapin_delay;             // swapin 延迟
    // CONFIG_TASK_DELAY_ACCT 时额外字段
    long freepages_delay;          // 内存回收延迟
    long thrashing_delay;          // thrashing 延迟
    long allocpage_delay;          // 页分配延迟
    long cpu_runtime;              // CPU 运行时间
    long cpu_run_delay;            // CPU 运行延迟
    int cpu;                       // 当前 CPU
    unsigned int freq, freq_max;   // CPU 频率
    // 睡眠/阻塞统计
    u32 sleep_cnt, sleep_tot, sleep_max;
    int wake_s_pid;
    u32 block_cnt, block_tot, block_max;
    int wake_d_pid;
} __attribute__((packed));
```
### 3.2 Native 库层 (libbperflog)

#### 3.2.1 线程本地 FD 管理

**设计思路：** 每个线程独立打开 `/proc/bperf/task_acct`，避免多线程竞争。

#### 3.2.2 BPerfLog 类核心方法

```c
class BPerfLog {
  public:
    bool blockSyncBegin(task_info& begin);     // 开始监控
    bool blockSyncEnd(task_info& end);         // 结束监控
    void blockSyncEndReport(string block, int64_t timeout, task_info& begin); // 带上报
  private:
    std::atomic<bool> mPlatSupport{true};      // 平台支持标记
};
```

#### 3.2.3 日志输出格式

```c
std::string get_task_info(task_info& begin, task_info& end) {
    // 输出格式示例：
    // tinfo=<running=100ms runnable=20ms blkio=50ms swapin=10ms
    //        free=5ms thrash=3ms alloc=2ms
    //        cpu=0,1200,1800:1,800,1800
    //        wake_s=5,100ms,50ms,1234
    //        wake_d=3,200ms,100ms,5678>
}
```

### 3.4 Framework 层集成

#### 3.4.1 IPerfLog 接口

```java
public interface IPerfLog {
    // 阻塞同步方法
    void blockSyncBegin();
    void blockSyncEnd(String blockInfo, long timeout);

    // Looper/Input 专用方法
    void looperSyncBegin();
    void looperSyncEnd(Message msg);
    void inputSyncBegin();
    void inputSyncEnd(InputEvent event);

    // 直接 JNI 调用
    long[] markTaskBegin();
    long[] markTaskEnd();
    void markTaskEndReport(String blockInfo, long timeout, long[] delayArr);

    // 预定义阈值
    int TASK_THREASHOLD_DOFRAME_MS = 24;
    int TASK_THREASHOLD_LOOPER_MS = = 1000;
    int TASK_THREASHOLD_BROADCAST_MS = 800;
    // ...
}
```

#### 3.4.3 各模块集成点

| 模块 | 监控点 | 阈值 |
|------|--------|------|
| **ActivityThread** | handleMessage 开始/结束 | 750ms |
| **BroadcastReceiver** | onReceive 开始/结束 | 800ms |
| **Activity onResume** | resume 调用 | 600ms |
| **Activity onPause** | pause 调用 | 600ms |
| **ContentProvider** | onCreate |  1000ms |
| **Looper** | 消息分发 | 1000ms |
| **Choreographer** | doFrame | 24ms |
| **InputEventReceiver** | 输入事件处理 | 350ms |
| **hwui DrawFrameTask** | 渲染帧 | 24ms |
| **SurfaceFlinger** | 帧合成消费 | 24ms |
| **Binder (IPCThreadState)** | 同步 binder 调用 | 600ms |

---

## 四、解决的核心问题

  1. 性能瓶颈定位困难
                                                                                                                 
  传统日志只有时间戳，无法知道耗时来自哪里。BPerfLog 提供细粒度延迟分解：
  
  tinfo=<running=500ms runnable=100ms blkio=150ms swapin=30ms free=10ms ...>
  
  一眼看出：              
  - running=500ms → CPU 执行慢（代码问题）
  - blk=150ms → 等待磁盘 I/O（读取 APK/DEX）
  - free=10ms → 内存回收延迟（内存压力）

  2. Binder/IPC 调用慢无法追踪

  Binder 调用耗时 600ms+，不知道是服务端慢还是调度慢。系统自动监控：

  perflog: hwbinder caller=5678 walltime=650ms
           tinfo=<running=200ms blkio=300ms runnable=50ms>

  - blkio=300m → 服务端在做 I/O
  - running=200ms → 服务端 CPU 处理

  3. 帧丢失原因不明

  应用丢帧只知道"渲染慢"，不知道是绘制慢还是调度慢：

  perflog: doframe walltime=30ms (超过 24ms 阈值)
           tinfo=<running=25ms runnable=3ms>

  - running=25ms → 主线程绘制逻辑慢
  - runnable=3ms → CPU 调度影响小
   4. 阻塞链追溯

  知道"谁唤醒了我"和"谁阻塞了我"：

  wake_s_pid=1234  // 最大睡眠时间对应的唤醒者 PID
  wake_d_pid=5678  // 最大阻塞时间对应的唤醒者 PID

  可以追溯整个阻塞链，找到根本原因。

  ---

## 九、总结

BPerfLog 是一个完整的性能监控系统，从内核驱动到 Framework 层紧密协作：

- **内核层**：通过 tracepoint hook 采集任务调度、I/O、内存等延迟数据
- **Native 层**：提供线程本地 FD 管理、ioctl 封装、日志格式化
- **JNI 层**：桥接 Java 与 Native，处理数据数组转换
- **Framework 层**：在关键路径插入监控点，自动上报超时情况
