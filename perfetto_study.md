perfetto
面试中经常问到的问题便是你怎么使用perfetto去查看性能问题，笔者对这方面也只是会基础的使用，如：看线程状态（running/io等），看线程消耗算力，某个线程被阻塞原因，sql语句的使用，数据源的选择等。
笔者自认为这些技巧可以应对使用中的大部分问题，没想到面试还是直接g了，后面系统的学习一下使用方法。
### 一些网站:
https://www.cnblogs.com/joy99/p/19755058
https://androidperformance.com/2024/05/21/Android-Perfetto-01-What-is-perfetto/#Perfetto-%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95
https://zhuanlan.zhihu.com/p/375503767

### 1.perfetto全过程：从traced进程启动->ftrace/atrace数据源获取->java/native层埋点->命令行抓取perfetto->分析perfetto

### 2.命令行抓取perfetto：
在 Android 12 及之后的设备上
从 Android 12 开始，可以直接使用/data/misc/perfetto-configs目录来存储配置文件，这样就不需要通过 stdin 来传递配置文件了。具体命令如下：

adb push config.pbtx /data/misc/perfetto-configs/config.pbtx
adb shell perfetto --txt -c /data/misc/perfetto-configs/config.pbtx -o /data/misc/perfetto-traces/trace.perfetto-trace
在这个例子中，首先将配置文件config.pbtx推送到/data/misc/perfetto-configs目录中。然后，直接在 Perfetto 命令中通过-c选项指定配置文件的路径来启动跟踪。

在 Android 12 之前的设备上
由于 SELinux 的严格规则，直接通过文件路径传递配置文件在非 root 设备上会失败。因此，需要使用标准输入(stdin)来传递配置文件。这可以通过将配置文件的内容cat到 Perfetto 命令中实现。具体命令如下：

adb push config.pbtx /data/local/tmp/config.pbtx
adb shell 'cat /data/local/tmp/config.pbtx | perfetto -c - -o /data/misc/perfetto-traces/trace.perfetto-trace'
这里，config.pbtx是你的 Perfetto 配置文件，首先使用adb push命令将其推送到设备的临时目录中。然后，使用cat命令将配置文件的内容传递给 Perfetto 命令

### 3.分析perfetto
#### 3.1 基础概念
1） 查看被哪个线程唤醒：点具体trace段cpu信息，查看黑点所在位置 ，Scheduling Latency的计算方式是：sched_switch时间 - sched_wakeup(sched_waking)时间

2） 依赖链：Critical Path 

3） 掉帧，可以看expected time 和实际时间，还有日志等。在安卓12上，日志与frametimeline和ftrace并不兼容，只能抓到一方的数据，附一份trace配置文件
