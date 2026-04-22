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
  - 2 rendernode 的create 时长 为4ms，上个版本只有 5微秒 -》查看对应修改，显示组有一笔修改，关于高斯模糊从gpu合成变成device合成。发现在修改中，将所有的rendernode都编译一个组件并进行初始化 -》要求显示进行按需使用该组件，或者换方案

### 3.仪表简易模式
bg：仪表在开机后进入简易模式

action：
- 无仪表堆栈。排查系统资源使用情况，发现io操作较慢，系统较多应用卡在io相关操作。查看内核日志，发现io挂载失败，转给io分析
- TODO
