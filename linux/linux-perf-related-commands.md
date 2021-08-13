几个linux性能分析工具

## dstat
dstat 一个用来替换vmstat, iostat, netstat, nfsstat 和ifstat 这些命令的工具，是一个全能系统信息统计工具
## mpstat
```
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
```
## vmstat
## top
## htop
## sar
sar -r 1 5 报告内存使用率的统计数据. 可以列出系统在各个时间的SWAP使用情况 (free 和sar显示的都不是实时数据)
```
#sar 2 5 同mpstat
Linux 4.4.0-21-generic (R27-IDP-8)      06/07/2016      _x86_64_        (12 CPU)
 02:32:49 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
 02:32:51 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:53 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:55 AM     all      0.00      0.00      0.00      0.04      0.00     99.96
 02:32:57 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
 02:32:59 AM     all      0.00      0.00      0.00      0.00      0.00    100.00
Average:        all      0.00      0.00      0.00      0.01      0.00     99.99
```
## free -m 提供概要信息
``` 
              total        used        free      shared  buff/cache   available
Mem:         128897         335      125615           9        2947      128184
Swap:        131025           0      131025
```
## pmap
显示各种进程分别占用内存情况
# slabtop

## network
- netstat
- tcpdump
- wireshark
- iptraf
- iftop

## disk/io
- iostat
```
# iostat -c 2 10  查看所有CPU的平均信息
Linux 4.4.0-21-generic (R27-IDP-8)      06/07/2016      _x86_64_        (12 CPU)
 avg-cpu:  %user   %nice %system %iowait  %steal   %idle
            0.00    0.00    0.00    0.00    0.00   99.99
 Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               0.15         0.21         5.44     217402    5716828
```
- iptop
## numa
- numactl --hardware
- numastat 查看每个node的内存统计
- numastat -c qemu-system-x86 查看每个节点的内存统计
```
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
```
- numad //apt install  numad  
- numatune 使用numatune 命令查看或者修改虚拟机的numa配置
```
get numa parameters:
virsh # numatune vm-01
 numa_mode      : strict
numa_nodeset   :
explain:
```
- pstree
- strace  查看进行调用系统的情况