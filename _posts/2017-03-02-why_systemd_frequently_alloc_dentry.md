## 问题

开启sysfs中dentry的trace：  
\# echo 1 > /sys/kernel/slab/dentry/trace  
\# echo 0 > /sys/kernel/slab/dentry/trace  

发现systemd在频繁分配dentry：  

\# grep -A1 "TRACE dentry alloc" /var/log/messages  | grep "Comm: systemd-journal" | wc -l  

 3763  

##分析过程

1，编写systmtap脚本，跟踪d_alloc：  
\# cat trace_systemd_dentry.stp  
\#!/usr/bin/stap  
probe begin  
\{  
    printf("ready !\n");  
\}  

probe kernel.function("d_alloc")  
\{  
	if(pid2execname(pid()) == "systemd-journal"){  
		name1 = @cast($name, "qstr", "kernel")->name;  
		print("\n");  
		printf(">>> pid[%d] pid_name[%s] \n", pid(), pid2execname(pid()));  
		printf("parentname [%s], dname[%s]\n", d_name($parent), kernel_string(name1) );  
		printf("parentname [%s], dname[%s]\n",  
				reverse_path_walk($parent),  
			       	kernel_string(name1));  
		print_backtrace();  
		print_ubacktrace();  
		print("\n");  
\	}  
\}  


2,开始跟踪：  

stap trace.stp -d /usr/lib64/libpthread-2.17.so  -d /usr/lib64/libc-2.17.so  -d /usr/lib/systemd/systemd-journald  
3，开启dentry的trace之后，很快频繁打印：  
>>> pid[605] pid_name[systemd-journal]  
parentname [/], dname[vz]  
 0xffffffff811cf3c0 : d_alloc+0x0/0x70 [kernel]  
 0xffffffff811c118b : lookup_dcache+0x8b/0xb0 [kernel]  
 0xffffffff811c11dd : __lookup_hash+0x2d/0x60 [kernel]  
 0xffffffff815eb5df : lookup_slow+0x4c/0xb1 [kernel]  
 0xffffffff811c3b08 : path_lookupat+0x7a8/0x7e0 [kernel]  
 0xffffffff811c3b6b : filename_lookup+0x2b/0xc0 [kernel]  
 0xffffffff811c7a17 : user_path_at_empty+0x67/0xd0 [kernel]  
 0xffffffff811c7a91 : user_path_at+0x11/0x20 [kernel]  
 0xffffffff811b4d02 : sys_faccessat+0xb2/0x230 [kernel]  
 0xffffffff811b4e98 : sys_access+0x18/0x20 [kernel]  
 0xffffffff815ff187 : tracesys+0xdd/0xe2 [kernel]  
 0x7fcf8f4a1b07 : access+0x7/0x30 [/usr/lib64/libc-2.17.so]  
 0x7fcf90cc0b3e : detect_container+0x8e/0x1c0 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cb6ead : audit_loginuid_from_pid+0x3d/0x130 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cb148d : dispatch_message_real+0x5fd/0x1d30 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cb2e24 : server_driver_message+0x264/0x420 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cae5ef : server_read_dev_kmsg+0x78f/0x920 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cb3fc5 : process_event+0x355/0x870 [/usr/lib/systemd/systemd-journald]  
 0x7fcf90cac304 : main+0x1e4/0x300 [/usr/lib/systemd/systemd-journald]  
 0x7fcf8f3dbb15 : __libc_start_main+0xf5/0x1c0 [/usr/lib64/libc-2.17.so]  
 0x7fcf90cac45d : _start+0x29/0x2c [/usr/lib/systemd/systemd-journald]  
在频繁访问/proc/vz 这一个文件：  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.056606] TRACE dentry alloc 0xffff88001ac16240 inuse=21 fp=0x          (null)  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.056609] CPU: 3 PID: 605 Comm: systemd-journal Tainted: GF          O--------------   3.10.0-123.el7.x86_64 #1   
--  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.056728] TRACE dentry free 0xffff88001ac16240 inuse=7 fp=0xffff88001ac16480  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.056730] Object ffff88001ac16240: 00 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00  ................   
--  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.057180] TRACE dentry alloc 0xffff88001ac16240 inuse=21 fp=0x          (null)   
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.057183] CPU: 3 PID: 605 Comm: systemd-journal Tainted: GF          O--------------   3.10.0-123.el7.x86_64 #1  
--  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.057257] TRACE dentry free 0xffff88001ac16240 inuse=7 fp=0xffff88001ac16b40  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.057259] Object ffff88001ac16240: 00 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00  ................  
--  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.058734] TRACE dentry alloc 0xffff88001ac16240 inuse=21 fp=0x          (null)  
Apr 12 18:16:52 SDN-229-1 kernel: [ 8197.058738] CPU: 3 PID: 605 Comm: systemd-journal Tainted: GF          O--------------   3.10.0-123.el7.x86_64 #1  

它频繁打开、关闭同一个文件，更改它的引用计数，不会导致dentry只增不减。  
dev_kmsg_record函数：  
                /* Did we lose any? */  
                if (serial > *s->kernel_seqnum)  
                        server_driver_message(s, SD_MESSAGE_JOURNAL_MISSED, "Missed %"PRIu64" kernel messages",  
                                              serial - *s->kernel_seqnum);  
打开dentry的trace，日志打印暴增很多，当出现日志丢失的时候，才会进入这个分支。  

另外，分析代码，在audit_session_from_pid函数中频繁访问/proc/vz这的逻辑在高版本systemd中已经去掉了：  

commit db999e0f923ca6c2c1b919d0f1c916472f209e62  
Author: Lennart Poettering <lennart@poettering.net>  
Date:   Wed Feb 12 02:52:39 2014 +0100  

    nspawn: newer kernels (>= 3.14) allow resetting the audit loginuid, make use of this  

diff --git a/src/shared/audit.c b/src/shared/audit.c  
index 8038ac3..5466447 100644  
--- a/src/shared/audit.c  
+++ b/src/shared/audit.c  
@@ -42,10 +42,6 @@ int audit_session_from_pid(pid_t pid, uint32_t *id) {  

         assert(id);  

\-        /* Audit doesn't support containers right now */  
\-        if (detect_container(NULL) > 0)  
\-                return -ENOTSUP;  
\-  
         p = procfs_file_alloca(pid, "sessionid");  
  
         r = read_one_line_file(p, &s);  
@@ -71,10 +67,6 @@ int audit_loginuid_from_pid(pid_t pid, uid_t *uid) {  
  
         assert(uid);  
  
\-        /* Audit doesn't support containers right now */  
\-        if (detect_container(NULL) > 0)  
\-                return -ENOTSUP;  
\-  
         p = procfs_file_alloca(pid, "loginuid");  
  
         r = read_one_line_file(p, &s);  




 
