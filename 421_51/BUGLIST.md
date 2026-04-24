### 1.gpu负载问题
bg:语音唤醒时，发现全景画中画出现掉帧现象

action：抓取perfetto进行分析，查看surfaceflinger的buffer在这一刻的buffer数量为0，全景的buffer数量也为0。查看全景渲染线程出现一下三种类型
- 1.rendenthread 出现长时间空白 -》camera数据传递给全景较慢-》camera数据卡在hwbinder -》提高hwbinder的优先级
- 2.renderthread cpu 长时间处于 runnable状态 -》渲染线程 加入 ux队列
- 3.renderthread 在长时间等待gpu -》 gpu负载问题，抓该场景各模块gpu负载数据，得到结论 全景gpu较高，需要优化

### 2.无人机卡顿问题
bg：无人机apk在新系统版本出现卡顿现象

action： 
- 抓取两个系统版本该apk对应界面的perfetto，发现该apk在满频大核上持续运行，非系统资源问题。可以确认该apk在新旧版本上running时间相差10倍是该界面流程问题。
- 抓取火焰图进行比较分析，发现两个疑点：
  - 1 系统组件频繁打印堆栈，大概占系统启动一半时间 -》经过了解，这是在打印堆栈日志-》要求优化日志或取消该方法
  - 2 rendernode 的create 时长 为4ms，上个版本只有 5微秒 -》查看对应修改，显示组有一笔修改，基于hwui的控件级液态玻璃链路api。发现在修改中，将所有的rendernode都编译一个组件并进行初始化 -》要求显示进行按需使用该组件，或者换方案

### 3.仪表简易模式
bg：仪表在开机后进入简易模式

action：
- 无仪表堆栈。排查系统资源使用情况，发现io操作较慢，系统较多应用卡在io相关操作。查看内核日志，发现io挂载失败，转给io分析
- TODO
### 4.应用内存增加问题
bg： 
- 1. 问题的发现：近期特性上主干，研发/测试 批量切换到主干系统版本，问题批量被发现。
- 2. 显示占用Native Heap  并且无法释放
- 3. 发生在DiLink150 主干版本，与其他平台差异是什么？为什么其他平台/量产版本 没有这个问题
- 4. DiLink150 平台主干版本 新音乐/旧音乐/微信/抖音 APP，内存都占用较大
- 5. 音乐重写，策略“返回键并不会销毁Activity”； debug 测试：测试Activity销毁的场景（内存回收）
  - 调试：如果 Activity#onDestory 能回收，问题不严重
  - 问题：验收过程中，体现的很卡&观察到PSS爆炸。
  - 期望：前台页面存在时，依旧不会出现内存过大的问题。
- 6. DILink150  最新主干版本 checkout from 量产版本，不是近期改出来的

TODO：
  - Debug Activity#onDestroy 是否可以回收该部分内存，对比其他平台 onDestroy 的内存回收效果
  - 分析 Java Heap/Native Heap 继续分析，找出线索
  - 分析 native Heap 回收机制，对比DiLink150主干与其他平台 显示链路 内存Native占用的差异

问题解决
- 问题原因：因为给每个控件都加了effect(每个 View 都有一个 RenderNode，每个 RenderNode 都有一个 RenderProperties，所以 每个 View 都会额外分配一个 BydAlgModule 对象，不管是否使用液态玻璃效果)，占用内存比较大，最新主干代码已经去掉
- 问题解决：修改 /system/lib64/libhwui.so，移除非必要的effect，20260421 后的主干版本修复了该问题
