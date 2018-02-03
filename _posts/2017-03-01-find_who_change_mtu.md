---
layout: post
title: 跟踪哪个进程在频繁更改系统时间
---

## 问题

message中Mar 29 17:35:54 localhost systemd: Time has been changed
打印非常非常频繁,怎么找到是哪个进程在调用？

-----

## 方法

* autitctl监控

auditctl -a exit,always -S adjtimex  -b64 -k XXX_adj
auditctl -a exit,always -S settimeofday  -b64 -k XXX_set
---》没有效果。


考虑到是系统调用没有监控全，使用ftrace：

* ftrace监控

cd    /sys/kernel/debug/tracing/events/syscalls

[root@localhost syscalls]# echo 1 > sys_enter_adjtimex/enable

[root@localhost syscalls]# echo 1 > sys_enter_clock_settime/enable

[root@localhost syscalls]# echo 1 > sys_enter_clock_adjtime/enable

[root@localhost syscalls]# echo 1 > sys_enter_settimeofday/enable


cd    /sys/kernel/debug/tracing/

 #watch -n 1 cat trace
 
 ClockSynTaskEnt-15461 [005] .... 33864.664181: sys_clock_settime(which_clock: 0, tp: 7f12a1a72ae0)
 
 ClockSynTaskEnt-15461 [005] .... 33884.671568: sys_clock_settime(which_clock: 0, tp: 7f12a1a72ae0)
 
 ClockSynTaskEnt-15461 [005] .... 33904.679034: sys_clock_settime(which_clock: 0, tp: 7f12a1a72ae0)
 
 ---》观察到是 ClockSynTaskEnt-15461 进程在频繁调用sys_clock_settime 更改时间。
