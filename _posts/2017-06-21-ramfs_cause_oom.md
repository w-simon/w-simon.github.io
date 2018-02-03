---
layout: post
title: Ramfs占用过大导致OOM
---

## 问题

系统发生OOM，系统日志如下：

[ 3994.943403] active_anon:1916 inactive_anon:7318 isolated_anon:32

                active_file:311 inactive_file:2079 isolated_file:0
				
                unevictable:15361039 dirty:0 writeback:0 unstable:0
				
                free:59725 slab_reclaimable:42227 slab_unreclaimable:24575
				
                mapped:300389 shmem:465 pagetables:3476 bounce:0
				
                free_cma:0
				
[ 3994.943407] Node 1 Normal free:45212kB min:45348kB low:56684kB high:68020kB active_anon:2380kB inactive_anon:8144kB active_file:64kB inactive_file:328kB unevictable:30862812kB isolated(anon):0kB isolated(file):0kB present:134217728kB managed:32501996kB mlocked:7376kB dirty:0kB writeback:0kB mapped:180756kB shmem:4kB slab_reclaimable:77140kB slab_unreclaimable:30920kB kernel_stack:2896kB pagetables:1432kB unstable:0kB bounce:0kB free_cma:0kB writeback_tmp:0kB pages_scanned:47491 all_unreclaimable? yes

[ 3994.943413] lowmem_reserve[]: 0 0 0 0

[ 3994.943415] Node 1 Normal: 471*4kB (UEM) 363*8kB (UEM) 222*16kB (UEM) 176*32kB (UEM) 135*64kB (EM) 65*128kB (UM) 30*256kB (UM) 5*512kB (UM) 0*1024kB 0*2048kB 1*4096kB (R) = 45268kB

[ 3994.943427] Node 0 hugepages_total=95 hugepages_free=2 hugepages_surp=0 hugepages_size=1048576kB

[ 3994.943429] Node 1 hugepages_total=95 hugepages_free=87 hugepages_surp=0 hugepages_size=1048576kB

[ 3994.943430] 15250101 total pagecache pages

[ 3994.943432] 4916 pages in swap cache

[ 3994.943433] Swap cache stats: add 528400, delete 523484, find 100279/115359

[ 3994.943434] Free swap  = 3604956kB

[ 3994.943435] Total swap = 4194300kB

[ 3994.943437] 67073695 pages RAM

[ 3994.943438] 0 pages HighMem/MovableOnly

[ 3994.943438] 50927604 pages reserved

## 分析

系统日志中，发现unevictable 占用的内存太大了（unevictable:15361039）。

解析vmcore：

crash> kmem -i

                 PAGES        TOTAL      PERCENTAGE
				 
    TOTAL MEM  65953451     251.6 GB         ----
	
         FREE    62432     243.9 MB    0% of TOTAL MEM
		 
         USED  65891019     251.4 GB   99% of TOTAL MEM
		 
       SHARED   314014       1.2 GB    0% of TOTAL MEM
	   
      BUFFERS      200       800 KB    0% of TOTAL MEM
	  
       CACHED  15243221      58.1 GB   23% of TOTAL MEM
	   
         SLAB    65466     255.7 MB    0% of TOTAL MEM

   TOTAL SWAP  1048575         4 GB         ----
   
    SWAP USED   129902     507.4 MB   12% of TOTAL SWAP
	
    SWAP FREE   918673       3.5 GB   87% of TOTAL SWAP

 COMMIT LIMIT  9121620      34.8 GB         ----
 
    COMMITTED   641419       2.4 GB    7% of TOTAL LIMIT
	
crash> 

 CACHED  15243221 和 unevictable:15361039 大小对的上。
 
 cached + unevictable 怀疑是ramfs占据的内存。
 
分析ramfs的挂载点：

crash> mount

     MOUNT           SUPERBLK     TYPE   DEVNAME   DIRNAME
	 
	 ......
	 
ffff8807aedff700 ffff883fd0492800 hugetlbfs hugetlbfs /dev/xxxxx

ffff8807d0a66e00 ffff883fd0717800 ramfs  ramfs     /var/lib/libvirt/qemu/ram

hugetlbfs 占用190G。

crash> p hstates

hstates = $3 = 

 {{
    next_nid_to_alloc = 0, 
	
    next_nid_to_free = 0, 
	
    order = 18, 
	
    mask = 18446744072635809792,
	
    max_huge_pages = 190,
	
    nr_huge_pages = 190,
	
    free_huge_pages = 20,
	
    resv_huge_pages = 20, 
	
	
分析ramfs占用多少内存：

crash> super_block ffff883fd0717800

struct super_block {

  s_list = {
  
    next = 0xffffffff819d4790 <super_blocks>, 
	
    prev = 0xffff883fd0492800
	
  }, 
  s_dev = 39, 
  
  s_blocksize_bits = 12 '\f', 
  
  s_blocksize = 4096, 
  
  s_maxbytes = 9223372036854775807, 
  
  s_type = 0xffffffff819e4200 <ramfs_fs_type>, 
  
  s_op = 0xffffffff8168c980 <ramfs_ops>, 
  
  dq_op = 0x0, 
  
  s_qcop = 0x0, 
  
  s_export_op = 0x0, 
  
  s_flags = 1610612736, 
  
  s_magic = 2240043254, 
  
  s_root = 0xffff88076ed3afc0, 
  
  s_umount = {
  
    count = 0,
	
    wait_lock = {
	
      raw_lock = {
	  
        {
		
          head_tail = 0,
		  
          tickets = {
		  
            head = 0,
			
            tail = 0
			
          }
        }
      }
    }, 
    wait_list = {
	
      next = 0xffff883fd0717878,
	  
      prev = 0xffff883fd0717878
	  
    }
  }, 
  s_count = 1, 
  s_active = {
  
    counter = 2
	
  }, 
  s_security = 0xffff883fd071b600, 
  s_xattr = 0x0, 
  s_inodes = {
    next = 0xffff8806d48ac1c8, 
    prev = 0xffff88076ee1d448
  }, 
  s_anon = {
    first = 0x0
  }, 
  s_files_deprecated = 0x0, 
  s_mounts = {
    next = 0xffff8807d0a66e60, 
    prev = 0xffff8807d0a66b60
  }, 

  
s_inodes是ramfs所有inode的链表，查询每个inode的size：

crash> list  -o  0x108  -H    0xffff8806d48ac1c8  -s inode.i_size

ffff8806d48ad7e0
  i_size = 1073741824
  
ffff8806d48ae120
  i_size = 1073741824
  
ffff8806d48a84a0
  i_size = 1073741824
  
ffff8806d48a9bc0
  i_size = 1073741824
  
ffff8806d48ab530
  i_size = 1073741824
  
ffff8806d48af5f0
  i_size = 1073741824
  
ffff8806d48ab780
  i_size = 1073741824
  
ffff8806d48aea60
  i_size = 1073741824
  
ffff8806d48ac7b0
  i_size = 1073741824
  
ffff8806d48aded0
  i_size = 1073741824
ffff8806d48ae810
  i_size = 1073741824
.......

发现ramfs占据58G，刚好也对应起来。

所以问题在于ramfs占据内存太大，ramfs使用的是pagecache的内存，但是会设置为dirty、unevictable，系统无法回收。

ramfs默认不会限制大小，如果一直增加会存在耗尽内存的风险。

修复方法是替换为tmpfs，并且限制大小。



