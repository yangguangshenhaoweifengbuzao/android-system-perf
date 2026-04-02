### 1.我的车桌面，卡片栏应用返回到桌面，出现卡顿
ActivityStartController.java中的startHomeActivity里面给launcher(主线程和渲染线程)加1s的ux
### 2.SR和地图分屏，拖动抖音，抖音小窗卡顿
给分屏相关线程设ux，FreeformTaskManager.java 给该handler设ux
### 3.set avc as ux When shifting into Drive for the first time
给avc所有线程加2sux，当挂D档的时候，在ipel服务初始化时注册回调，监听D档和accon
### 4.权限检查
只有白名单中的应用才可以主动调用ux接口
### 5.boost gpu frequency by input event(1/3)
onInputEvent方法里面提升gpu频率
### 6.:set BydSmartMultiDividerController thread and SystemServer RenderThread as ux
初始化时给分屏线程加ux，做动画时给renderthread线程加ux，退出动画取消ux
### 7. setLightState asynchronously during the boot process
通知线程提高优先级
### 8. [Title:for whitelisted app, speed compile for installing
设置编译速度
### 9.Title:don't set cpuset cgroup when lmkd kill processes
删除if (!SetProcessProfilesCached(uid, pid, profiles)) 判断
### 10.change compiler filter from speed-profile to speed for boot jars
增加参数cmd.FlagWithArg("--compiler-filter=", "speed")
### 11.Title:The touch input events skip inputfilter for inputflinger performance(1/1)
如果是触摸事件跳过filter判断
### 12.Title:cpu boost when launching app
cpu boost when launching app
### 13.delay snapshotTasks when app transition is ready
