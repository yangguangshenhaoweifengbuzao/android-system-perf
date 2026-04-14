## 1.性能负载上报

## 2. ActivityManagerService.java 和 ProcessRecord.java 中埋了三个hook：

 
     Hook点     |         位置             │                        触发事件                           
   进程启动    │ handleProcessStartedLocked() │ ProcessUtil.getInstance().procStart()                      
   进程退出    │ appDiedLocked()              │ ProcessUtil.getInstance().procExit()        
   可见性变化  │ ProcessRecord.setVisible()   │ ProcessUtil.getInstance().onProcessVisibilityChanged() 
   
## 3.配置系统

  XML配置文件 (powerconfig_Di100VCP.xml)：
  <item proc="进程名" bg="后台阈值" fg="前台阈值" limit="上限"/>
  <otherconfig interval="5000" max_upload_count="20">
      <item name="monitor_power" arraysize="3" value="7108,26239,29566"/>
  </otherconfig>

  阈值计算公式：阈值 × MONITOR_INTERVAL(1)

## 4.  数据流向
  
### 4.1  配置下发流程：

  PowerConfig读取XML → PowerGuardService.allReady()
  → JNI setDefaultPowerValue/setCpuPowerRate/setCheckInterval
  → ioctl(/dev/k_rs_warning) → Kernel

### 4.2  进程监控流程：

  AMS进程启动 → ProcessUtil.procStart()
  → 延迟3秒后 → PowerGuardService.startProc()
  → JNI setExcessivePowerValue → ioctl → Kernel设置该进程阈值

### 4.3  异常上报流程：

  Kernel检测到算力超限 → Netlink发送 rs_warn_cpu_payload
  → rs_netlink.c threadloop() 接收 → cpuload_state_cb回调
  → JNI调用 onCpuPowerExceeded(pid, cpuload)
  → PowerGuardService处理 → 上报数据（最多20次/进程）
### 4.4 关键机制

  1. 阈值动态调整：根据进程前后台状态切换不同阈值
  2. 上报次数限制：每进程最多上报MAX_UPLOAD_REPORT(20)次
  3. Native进程支持：通过读取/proc/[pid]/cmdline识别非Java进程
  4. 延迟启动监控：新进程延迟MONITOR_START_APP_DELAY(3000ms)才开始监控
