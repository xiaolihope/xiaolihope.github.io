link 涵盖了性能分析的命令: http://www.lxway.com/289440656.htm
and http://www.lxway.com/56118656.htm
系统优化必读: https://linux.cn/article-1769-1.html
#man dstat
SEE ALSO
    Performance tools
        ifstat(1), iftop(8), iostat(1), mpstat(1), netstat(1), nfsstat(1), nstat, vmstat(1), xosview(1)
    Debugging tools
        htop(1), lslk(1), lsof(8), top(1)
    Process tracing
        ltrace(1), pmap(1), ps(1), pstack(1), strace(1)
    Binary debugging
        ldd(1), file(1), nm(1), objdump(1), readelf(1)
    Memory usage tools
        free(1), memusage, memusagestat, slabtop(1)
   Accounting tools
       dump-acct, dump-utmp, sa(8)
    Hardware debugging tools
        dmidecode, ifinfo(1), lsdev(1), lshal(1), lshw(1), lsmod(8), lspci(8), lsusb(8), smartctl(8), x86info(1)
    Application debugging
        mailstats(8), qshape(1)
    Xorg related tools
        xdpyinfo(1), xrestop(1)
    Other useful info
       collectl(1), proc(5), procinfo(8)
system
$ uname -a
Linux R27-IDP-8 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
$ uname -r
4.4.0-21-generic
$ lsb_release -a
 No LSB modules are available.
 Distributor ID: Ubuntu
 Description:    Ubuntu 16.04 LTS
 Release:        16.04
Codename:       xenial
$ cat /etc/*ease
 DISTRIB_ID=Ubuntu
 DISTRIB_RELEASE=16.04
 DISTRIB_CODENAME=xenial
 DISTRIB_DESCRIPTION="Ubuntu 16.04 LTS"
 NAME="Ubuntu"
 VERSION="16.04 LTS (Xenial Xerus)"
 ID=ubuntu
 ID_LIKE=debian
 PRETTY_NAME="Ubuntu 16.04 LTS"
 VERSION_ID="16.04"
 HOME_URL="http://www.ubuntu.com/"
 SUPPORT_URL="http://help.ubuntu.com/"
 BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
UBUNTU_CODENAME=xenial
$ dmesg | grep numa
[    0.000000] mempolicy: Enabling automatic NUMA balancing. Configure with numa_balancing= or the kernel.numa_balancing sysctl
cpu
https://linux.cn/article-1770-1.html
# lscpu
Architecture:          x86_64
 CPU op-mode(s):        32-bit, 64-bit
 Byte Order:            Little Endian
 CPU(s):                12
 On-line CPU(s) list:   0-11
 Thread(s) per core:    1
 Core(s) per socket:    6
 Socket(s):             2
 NUMA node(s):          2
 Vendor ID:             GenuineIntel
 CPU family:            6
 Model:                 44
 Model name:            Intel(R) Xeon(R) CPU           X5670  @ 2.93GHz
 Stepping:              2
 CPU MHz:               1596.000
 CPU max MHz:           3059.0000
 CPU min MHz:           1596.0000
 BogoMIPS:              5866.91
 Virtualization:        VT-x
 L1d cache:             32K
 L1i cache:             32K
 L2 cache:              256K
 L3 cache:              12288K
 NUMA node0 CPU(s):     0-5
 NUMA node1 CPU(s):     6-11
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 cx16 xtpr pdcm pcid dca sse4_1 sse4_2 popcnt aes lahf_lm epb tpr_shadow vnmi flexpriority ept vpid dtherm ida arat
#mpstat
#vmstat
#top
#htop
# sar
memory
# free -m 提供概要信息
               total        used        free      shared  buff/cache   available
 Mem:         128897         335      125615           9        2947      128184
Swap:        131025           0      131025
# pmap  显示各种进程分别占用内存情况
# cat /proc/meminfo | grep 提供详细信息
# ps -aux
# top
# htop
// # gmemusage
// # memusagestat
# slabtop
http://www.lxway.com/58502224.htm
kswapd 和 pdflush
# sar -r 1 5 报告内存使用率的统计数据. 可以列出系统在各个时间的SWAP使用情况 (free 和sar显示的都不是实时数据)
# vmstat  
network
# netstat
# tcpdump
#wireshark
# iptraf
# iftop
disk/io
# iostat
link: http://www.lxway.com/944159852.htm
# iptop
numa
1)查看硬件信息
# numactl --hardware
available: 2 nodes (0-1)
 node 0 cpus: 0 1 2 3 4 5
 node 0 size: 64388 MB
 node 0 free: 62443 MB
 node 1 cpus: 6 7 8 9 10 11
 node 1 size: 64509 MB
 node 1 free: 63171 MB
 node distances:
 node   0   1
   0:  10  21
  1:  21  10
2)查看每个node的内存统计
# numastat
                            node0           node1
 numa_hit                 3780061         3373191
 numa_miss                      0               0
 numa_foreign                   0               0
 interleave_hit             39356           39803
 local_node               3765367         3347482
other_node                 14694           25709
libvirt 的numa 管理
3) 查看每个节点的内存统计
# numastat -c qemu-system-x86
 Per-node process memory usage (in MBs)
 PID              Node 0 Node 1 Total
 ---------------  ------ ------ -----
 14253 (qemu-syst    392    203   595
 14798 (qemu-syst    370    178   547
 ---------------  ------ ------ -----
Total               762    381  1143
# numastat -c vm-01
 Per-node process memory usage (in MBs) for PID 14253 (qemu-system-x86)
          Node 0 Node 1 Total
          ------ ------ -----
 Huge          0      0     0
 Heap          5     98   103
 Stack         0      0     0
 Private     369    122   492
 -------  ------ ------ -----
Total       375    221   595
# apt install  numad
# numad  
4) 使用numatune 命令查看或者修改虚拟机的numa配置
get numa parameters:
virsh # numatune vm-01
 numa_mode      : strict
numa_nodeset   :
explain:

ref: http://www.tuicool.com/articles/7FVR32Y
performance command
ref link: http://www.lxway.com/669869412.htm
查看各个进程cpu的使用率和内存使用率
# top
top - 02:08:43 up 12 days,  3:53,  3 users,  load average: 0.00, 0.01, 0.05
Tasks: 229 total,   1 running, 228 sleeping,   0 stopped,   0 zombie
 %Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
 KiB Mem : 13199153+total, 12862944+free,   343652 used,  3018452 buff/cache
 KiB Swap: 13417062+total, 13417062+free,        0 used. 13126006+avail Mem
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
10743 root      20   0   40704   3884   3168 R   0.3  0.0   0:00.01 top
TOP命令增加列显示:
执行top命令之后, 按f键,然后按空格选中需要增加的显示列. q退出
查看整个机器的CPU,内存,IO的使用情况
# vmstat 2 只能查看所有CPU的平均信息,查看CPU队列信息
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 128630080 153152 2865312    0    0     0     0    2    1  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0  106  129  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0   85   95  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0   82   84  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0   87   94  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0  100  127  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0   66   53  0  0 100  0  0
 0  0      0 128630192 153152 2865312    0    0     0     0  103  121  0  0 100  0  0
0  0      0 128630192 153152 2865312    0    0     0     0   72   57  0  0 100  0  0
# htop
# dstat   一个用来替换vmstat, iostat, netstat, nfsstat 和ifstat 这些命令的工具，是一个全能系统信息统计工具
# mpstat -P ALL( Multiprocessor Statistics  need install sysstat) 不但能查看所有CPU的信息,还能查看指定CPU的信息.
Linux 4.4.0-21-generic (R27-IDP-8)      06/07/2016      _x86_64_        (12 CPU)
 02:06:10 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
 02:06:10 AM  all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   99.99
 02:06:10 AM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   99.99
 02:06:10 AM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   99.99
 02:06:10 AM    5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM    6    0.00    0.00    0.00    0.02    0.00    0.00    0.00    0.00    0.00   99.97
 02:06:10 AM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   99.99
 02:06:10 AM    8    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM    9    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
 02:06:10 AM   10    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:06:10 AM   11    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
# iostat -c 2 10  查看所有CPU的平均信息
Linux 4.4.0-21-generic (R27-IDP-8)      06/07/2016      _x86_64_        (12 CPU)
 avg-cpu:  %user   %nice %system %iowait  %steal   %idle
            0.00    0.00    0.00    0.00    0.00   99.99
 Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               0.15         0.21         5.44     217402    5716828
#sar 2 5 同mpstat
Linux 4.4.0-21-generic (R27-IDP-8)      06/07/2016      _x86_64_        (12 CPU)
 02:32:49 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
 02:32:51 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:53 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:55 AM     all      0.00      0.00      0.00      0.04      0.00     99.96
 02:32:57 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:59 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
Average:        all      0.00      0.00      0.00      0.01      0.00     99.99
# uptime  查看平均负载,显示系统运行了多长时间, 和用户登陆数
02:27:32 up 12 days,  4:12,  3 users,  load average: 0.01, 0.02, 0.05
# ps 
# pstree
# strace  查看进行调用系统的情况
# 
libvirt tune 
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-NUMA-NUMA_and_libvirt.html
http://docs.openstack.org/developer/nova/testing/libvirt-numa.html

qemu-kvm 

Q: how to use qemu or virt-manager to deploy vm.
The relationship between virt-manager and libvrit.
 steps:
1) apt install install virt-manager
2) when staring virt-manager
    need install qemu-system
https://help.ubuntu.com/community/KVM/VirtManager
http://ubuntuforums.org/showthread.php?t=2203434
3) create vm remoted
    need copy the iso file to /var/lib/libvirt/images/ and restart virt-manager to get the iso image.
