1. memtester  memtest86+
压力测试; 捕获内存错误和一直处于很高或者很低的坏位.  进行的项目有: 随机值\异或比较\减法\乘法\除法\与或 给定测试内存的大小和次数.
link: http://my.oschina.net/guol/blog/59993
2. LMbench

3. STREAM
内存带宽
link: http://blog.csdn.net/maray/article/details/6230912
       http://blog.csdn.net/maray/article/details/6230912
       http://blog.chinaunix.net/xmlrpc.php?r=blog/article&amp;uid=28241199&amp;id=4233516
进行4中操作的测试: add, copy, triad(加/减/乘 结合起来), scale (乘法运算)

       
4.
apt-get install stress (a tool that can make the cpu stress to 100%)
5. mbw
用来测试应用程序进行内存拷贝操作所能达到的带宽. 
link: http://song-hope.blog.163.com/blog/static/37054942201211213419836/
1. setup environment:
2. run benchmarchs and analysis result:
LMbench, memtest86+, ... 
3. memory parameters
The draft plan of memory performance testing cases as below:
The purpose of the performance test:
* The performance degradation VM vs. HOST
* For multiply vms, test if it is fair in the matter of resource allocation.
* Find the best parameter value on Host or VM. 
The purpose of the parameters: 
* Increase the use ratio of memory.
ParametersHostVMNote
EPT/VPID
hardware support, anbled or disable in BIOS
to check if it enabled:
AMD/Intel
option1: cat /proc/cpuinfo 
option2: grep ept /proc/cpuinfo
cat /sys/module/kvm_intel/parameters/vpid
cat /sys/module/kvm_intel/parameters/ept
flags: ept vpid
Defaullt: enabledwe can enable it on vm
Transparnet Huge Page
check if enable:
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /proc/meminfo
Default: enabled
Both enabled by default.
swapping
how to set the value:
cat /proc/sys/vm/swapiness
Default: 60
Default: 60
It can be set to 0 on virtual vm
turn up: care concurrence amount do not care response time.
turn down: care response time, do not care concurrence amount.
ballooning 
Purpose: adjust the memory allocation betweenhost and guest vm.
need virtio_balloon
Advantage:
1)can be managed and controled,(different with ksm) 2) it is flexible, can apply little or much mememory. 3) ease memory pressure
Disadvantage:
1) Gust need install virtio_balloon driver 2) Would affect the gust performance when have the return memory request from host. 3) Would cause fragmentization, and the change of memory will affect the optimizing strategy in kernel.
How to use virtro_balloon:
scenario: 
There are 6 guest on a host. If we want overload memory between vms, we can use balloon.
manage tool:
libvirt
we can build the virtio_balloon driver in to image.
ksm
enable: we want run as mush as vms on host and do not care guest performance,
disable: we care about performance, and have enough memory.
fragmentize
cat /sys/kernel/mm/transparent_hugepage/defrag
both enabled by default.
cache policy
vm.dirty_background_ratio 5%
vm.dirty.ratio                       10%
turn down: when have continuous write operation runs.  eg db write
Environment:
*CPU+*GB HD +*GB RAM +*MB SWAP
tools:
1) LMbench
2) Memtest86+
3) STREAM
Test Scenario1: 
Tool: memtester(linux memory pressure test)
Read and write aim at the specified size of memory.  (lower is better)
test steps:
step1: install memtester
           download 
           tar 
           make 
           apt install gcc
memory size
time
20M200M2GHOSTVM1VM2
Test Scenario 2: 
Tool: Phoronix
HOSTVMHelpPhoronix Test Suite - CacheBenchMore is betterPhoronix Test Suite - RAMspeedMore is better
Test Scenario3:
tool: STREAM (memory bandwidth test)



4. steps
step1:  prepare a iso file on host. 
    met permission error when scp, the reason is the permision of the folder is not correct.
    $ chown dixiaoli iso
step2: install qemu related ...
    $ sudo apt-get install qume-kvm qemu virt-manager virt-viewer libvirt-bin
     met the issue: unable to locate the package.
    $ sudo apt-get update
     and run openvpn client.ovpn
 
