### 1.总述
在framework埋点，由native服务 perfmonitor 统一储存数据到本地，并将数据上报云端，云端制作性能看板
### 2.数据看板
- 1.流畅度-窗口帧率
```
  公共字段：car_brand=%s,car_model=%s,platform=%s,ui_ver=%s,dilink_ver=%s,build_ver=%s,mcu_ver=%s,vin=%s,iccid=%s,userid=%s,uptime=%s
  事件字段：scene=%s,dur=%d,fps=%.1f,proc=%s,uid=%d,win=%s,type=%d,time=%s
```
  bfps和perflog
- 2.时延-应用启动时间
```
  公共字段：car_brand=%s,car_model=%s,platform=%s,ui_ver=%s,dilink_ver=%s,build_ver=%s,mcu_ver=%s,vin=%s,iccid=%s,userid=%s,uptime=%s
  事件字段：proc=%s,uid=%s,act1=%s,act2=%s,type=%s,app_ver=%s,dtime=%s,time=%s
```
perflog
- 3.流畅度-卡顿事件
```
  公共字段：car_brand=%s,car_model=%s,platform=%s,ui_ver=%s,dilink_ver=%s,build_ver=%s,mcu_ver=%s,vin=%s,iccid=%s,userid=%s,uptime=%s
  事件字段：scene=%s,dur=%d,fps=%.1f,proc=%s,uid=%d,win=%s,type=%d,time=%s
```
janktracker
- 4.时延-anr事件
```
  公共字段：car_brand=%s,car_model=%s,platform=%s,ui_ver=%s,dilink_ver=%s,build_ver=%s,mcu_ver=%s,vin=%s,iccid=%s,userid=%s,uptime=%s
  事件字段：proc=%s,appver=%s,group=%s,type=%s,time=%s,e_log=%s
```
anrhelper
- 5.应用算力异常
