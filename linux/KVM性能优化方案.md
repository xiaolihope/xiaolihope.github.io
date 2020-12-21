1. KVM 性能优化
   CPU, NUMA
        (SMP;MPP;NUMA 是3中不同的CPU硬件体系架构
        SMP: 共享, 所有的CPU共享使用全部资源,内存 总线和IO
                 多个CPU 没有主次之分,平等地访问共享的资源, 势必引起资源竞争问题,导致扩展能力非常有限.
        MPP: 可以理解为SMP的横向扩展集群,MPP一般要依靠软件实现.
        NUMA: 将CPU分成不同的组(node), NUMA中的概念为:
            Node --> Socket --> Core --> Processor
            HT: Hyper-Threading 超线程. 一个core打开HT之后,在OS看来是2个核. 称为2个Logical Processor.
        )
        /proc/cpuinfo
        查看CPU topology:
            numactl 是设定进程NUMA 策略的命令行工具.
        numa工作方式可以是strict，指定cpu，或者auto 使用系统的numad服务
        绑定CPU执行的方法:
            1)  numactl -m 0 –physcpubind=2,3
                 numastat -c  qemu-kvm 查看每个节点的内存统计
                 numatune

            2)  taskset
            3) Pin, 将vm上的CPU 绑定到某一个node上. 让其共享L3-cache. 优先选择node上的内存, bind方法可以通过 virt-manage processor里面的pinning动态绑定。这个绑定是实时生效的
        Note1: 此方法仅仅进行CPU bind, 会共享l3-cache. 并没有限制一定使用某一个node上的内存,所以仍然会出现跨node使用内存的情况.
        Note2: NUMA 调优的目标就是让处理器尽量的访问自己的存储器，以提高处理速度

   内存,
        Cache : L1, L2, L3. L1 还分为独立的指令cache和数据cache.
        ls /sys/devices/system/cpu/cpu0/cache/
        index0            index1            index2       index3
        1级数据cache; 1级指令cache; 2级cache; 3级cache 对应cpuinfo中的cache
        优化项包括:
            EPT, 透明大页, 内存碎片整理, ksm(Kernel Samepage Merging),cache policy
            1. EPT(ExtendedPageTable):  vm vaddr----->vm padddr--------->host paddr , EPT的引入将两次地址转换变成了一次. 是在BIOS中,随着VT技术的开启一起开启的.
            2. 透明大页的开启：echo always > /sys/kernel/mm/transparent_hugepage/enabled
            3. 内存碎片整理的开启：echo always> /sys/kernel/mm/transparent_hugepage/defrag
            4. KSM:  简单理解就是可以将host机内容相同的内存合并，节省内存的使用，特别是当vm操作系统都一样的情况，肯定会有很多内容相同的内存，开启了KSM，则会将这些内存合并为一个，当然这个过程会有性能损耗，平均性能降低为10%， 所以开启与否，需要考虑使用场景，如果不注重vm性能，而注重host内存使用率，可以考虑开启，反之 则关闭，在/etc/init.d/下，会有两个服务，服务名称为ksmd和ksmtuned，都需要关闭。
                （rhel6 and fedora 14）默认是开启的.
            link: KSM: http://searchenterpriselinux.techtarget.com/tip/How-to-improve-KVM-performance-by-adjusting-KSM?utm_content=Linuxclusteract
                    KSM CN: http://www.361way.com/ksm-cpu-kvm/2756.html
                    *rhel 6 virtualiation : https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/chap-KSM.html
                    *kvm optimize: http://www.osforce.cn/course/77/material/

                判断是否enable ksm:
                    如果是为了能够run尽可能多的vm在host上，而不care性能的问题， 可以开启ksm, 否则，如果一个server上运行了数量比较少的vm，并且性能是问题， 应该关闭KSM。
                    最好的选择是： 根据内存情况来判断是否启用KSM。 如果再不起用KSM的情况下，RAM满足VM 的需求，则不需要开启。
                                            chkconfig ksmd off; chkconfig ksmtuned off; service ksmd off; service ksmtuned off
                                            如果host的内存紧张的话，最好开启KSM。
               调整KSM的参数，以达到最好的性能：
                $cat   /etc/ksmtuned.conf
               # Configuration file for ksmtuned.
# How long ksmtuned should sleep between tuning adjustments
# KSM_MONITOR_INTERVAL=60
# Millisecond sleep between ksm scans for 16Gb server.
# Smaller servers sleep more, bigger sleep less.
# KSM_SLEEP_MSEC=10 the most important parameter. better give a higher value when running a few VMs on the host.
# KSM_NPAGES_BOOST=300
# KSM_NPAGES_DECAY=-50
# KSM_NPAGES_MIN=64
# KSM_NPAGES_MAX=1250
# KSM_THRES_COEF=20
# KSM_THRES_CONST=2048
# uncomment the following if you want ksmtuned debug info
# LOGFILE=/var/log/ksmtuned
# DEBUG=1
            5.  NUMA Memory Policy  ***
                 https://www.kernel.org/doc/Documentation/vm/numa_memory_policy.txt
                 NUMA 的内存分配策略有localalloc、preferred、membind、interleave.
                       localalloc规定进程从当前node上请求 分配内存；
                       preferred比较宽松地指定了一个推荐的node来获取内存，如果被推荐的node上没有足够内存，进程可以尝试别的node
                       membind可以指定若干个node，进程只能从这些指定的node上请求分配内存
                       interleave规定进程从指定的若干个node上以RR算法 交织地请求分配内存
                       ref link: http://blog.chinaunix.net/uid-116213-id-3595888.html
                 查看是否支持numa:
                 $ dmesg | grep numa
                 查看numa的信息:
                 dixiaoli@R27-IDP-8:~$ numastat
                                                  node0           node1
numa_hit                 3622591         3201517
numa_miss                      0               0
numa_foreign                   0               0
interleave_hit             39356           39803
local_node               3607901         3175814
other_node                 14690           25703
numa_hit是打算在该节点上分配内存，最后从这个节点分配的次数;
num_miss是打算在该节点分配内存，最后却从其他节点分配的次数;
num_foregin是打算在其他节点分配内存，最后却从这个节点分配的次数;
interleave_hit是采用interleave策略最后从该节点分配的次数;
local_node该节点上的进程在该节点上分配的次数
other_node是其他节点进程在该节点上分配的次数
查看两个node 的cpu归属: $lscpu
$ numactl --hardware
查看各个cpu 负载情况:
$mpstat -P ALL(需要安装sysstat)
*** link: http://blog.csdn.net/jollyjumper/article/details/17168175
             6.  memory backing
                  https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-mem-back.html
<domain>
  ...
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
  ...
</domain>
            7.  Memory tuning
                 https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-libvirt-dom-xml-mem-tuning.html
<domain>
  ...
  <memtune>
    <hard_limit unit='G'>1</hard_limit>
    <soft_limit unit='M'>128</soft_limit>
    <swap_hard_limit unit='G'>2</swap_hard_limit>
    <min_guarantee unit='bytes'>67108864</min_guarantee>
  </memtune>
  ...
</domain>
            8.  Advanced Memory Management
http://www.linuxtopia.org/online_books/rhel6/rhel_6_virtualization/rhel_6_virtualization_ch22s02.html
            9. swappiness
                 swap 是将磁盘上的一部分空间用作交换，称之为虚拟内存。 当swappiness 越低就是叫OS尽量使用物理内存，然后才是SWAP空间。数值越高就算叫OS尽量使用SWAP
                 默认是60，也就算说内存使用到100-60=40%的时候，就开始出现有交换分区的使用。
                 By setting it to the maximum of 100 the kernel will swap very aggressively. By setting it to 0 the kernel will only swap to protect against an out-of-memory condition. The default is 60 which means that some swapping will occur.
                 Swapping is bad in a virtual environment, at any level. When the guest OS starts swapping on a physical server it only affects that server, but in a virtual environment it causes problems for all the workloads.
                 How to stop swap?
                 1. $ sudo sysctl -w vm.swappiness=0
                 2.  appending “vm.swappiness = 0” to file /etc/sysctl.conf and sudo sysctl –p to reload the values.
                 3. change it temporary cat /proc/sys/vm/swappiness
                     60
                 link: https://lonesysadmin.net/2013/12/11/adjust-vm-swappiness-avoid-unneeded-disk-io/
                 Note: 最新的linux内核中，如果设置此值为0，将会产生内存溢出的问题。 eg. 当内存溢出的时候，会kill掉消耗内存很大的cpu.
                     link: http://blog.csdn.net/longxibendi/article/details/38146521

                 所以如果希望提高服务器的并发量，对服务的相应时间要求不很高的场景可以适当的把swappniess调节的高些。对于并发量不大但希望相应时间小的应用场景可以适当的调小这个参数，比如个人电脑可以直接禁掉swap。
           10. RAM disk
                  RAM disk,(buffer/cache) and it trades some local memory for disk I/O
           11. 文件系统缓存策略cache policy  vm.dirty_background_ratio and vm.dirty_ratio
                  dirty: 是指需要写回磁盘的数据大小
                  vm.dirty_background_ration: 这个参数指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如5%）就会触发pdflush/flush/kdmflush等后台回写进程运行，将一定缓存的脏页异步地刷入外存；
                  vm.dirty_ratio: 而这个参数则指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如10%），系统不得不开始处理缓存脏页（因为此时脏页数量已经比较多，为了避免数据丢失需要将一定脏页刷入外存）；在此过程中很多应用进程可能会因为系统转而处理文件IO而阻塞。
                  link: blog.sina.com.cn/s/blog_448574810101k1va.html
                  $ sysctl -a | grep dirty
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 5  // 控制pdflush 进程在何时刷新磁盘。
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000 //这个参数表示page cache中的数据多久标记为脏
vm.dirty_ratio = 10  // 表示当写缓存使用到系统内存的多少时，pdflush开始向磁盘写出数据。 增大之，可以使更多的系统内存用于磁盘写缓冲，可以极大提高系统的写性能。 但是当需要持续、恒定的写入时，应该降低其数值。
vm.dirty_writeback_centisecs = 500 // 这个参数调节pdflush被唤醒的频率
可以运行sync命令，系统就会唤醒pdflush直至所有的脏页都写到磁盘。这是meminfo的dirty 就降下来了。
# cat /proc/vmstat | egrep "dirty|writeback"
nr_dirty 494
nr_writeback 0
nr_writeback_temp 0
nr_dirty_threshold 123214
nr_dirty_background_threshold 6160
directory: /proc/sys/vm
need to add the setting to /etc/sysctl.conf to make the changes to vm permanently.
link: https://lonesysadmin.net/2013/12/22/better-linux-disk-caching-performance-vm-dirty_ratio/
link: http://stackoverflow.com/questions/27900221/difference-between-vm-dirty-ratio-and-vm-dirty-background-ratio
link: http://www.cyberciti.biz/faq/linux-kernel-tuning-virtual-memory-subsystem/
***link: https://sites.google.com/site/sumeetsingh993/home/experiments/dirty-ratio-and-dirty-background-ratio
    note: need re-read!!
pdflush/kdmflush
link: http://blog.sina.com.cn/s/blog_96757e4b01011b1n.html
/proc/sys/vm/nr_pdflush_threads
test: http://labs.isee.biz/index.php/Board_validation_and_diagnostic_tools
12. 测试工具：
vmstat:
link : blog.163.com/xychenbaihu@yeah/blog/static/13222965520123724410678/

   磁盘,
       优化包括:  virtio-blk、缓存模式、aio、块设备io调度器
           virtio: 半虚拟化设备,可以在libvirt xml中设置，disk中加入<target dev='vda' bus='virtio'/>
           缓存模式:  从vm写磁盘，有3个缓冲区，guest fs page cache、Brk Driver writeback cache（qemu的cache）、Host FS page cache
                         启用方式:  libvirt xml disk中加入<driver name='qemu' type='qcow2' cache='none'/>
            aio:  异步读写，分别包括Native aio: kernel AIO 和 threaded aio: user space AIO emulated by posix thread workers，内核方式要比用户态的方式性能稍好一点，所以一般情况都选择native，开启方式<driver name='qemu' type='qcow2' cache='none' aio='native'/>
            块设备调度器:  cfq：perprocess IO queue，较好公平性，较低aggregate throughput
deadline：per-device IO queue，较好实时性，较好aggregate throughput，不够公平，当某些vm有大量io操作，占用了大量io资源时，其它后加入的vm很有可能抢占不到io资源。
这个目前笔者还没有做过测试，但是查看网易和美团云的方案，都将其设置为cfq。
开启方式：echo cfq > /sys/block/sdb/queue/scheduler
   网络
       优化包括: virtio、vhost、macvtap、vepa、SRIOV 网卡
           virtio:  更改虚拟网卡的类型，由全虚拟化网卡e1000、rtl8139，转变成半虚拟化网卡virtio，virtio需要qemu和vm内核virtio驱动的支持，这个原理和磁盘virtio原理一样.
           vhost_net: vhost_net将virtiobackend处理程序由user space转入kernel space，将减少两个空间内存拷贝和cpu的切换，降低延时和提高cpu使用率
           macvtap:  代替传统的tap+bridge，有4种模式，bridge、vepa、private、passthrough
               1, Bridge, 完成与 Bridge 设备类似功能，数据可以在属于同一个母设备的子设备间交换转发. 当前的Linux实现有一个缺陷，此模式下MACVTAP子设备无法和Linux Host通讯，即虚拟机无法和Host通讯，而使用传统的Bridge设备，通过给Bridge设置IP可以完成。但使用VEPA模式可以去除这一限制. macvtap的这种bridge模式等同于传统的tap+bridge的模式.
               2, VEPA, 式是对802.1Qbg标准中的VEPA机制的部分软件实现，工作在此模式下的MACVTAP设备简单的将数据转发到母设备中，完成数据汇聚功能，通常需要外部交换机支持Hairpin模式才能正常工作。
               3, Private, Private模式和VEPA模式类似，区别是子 MACVTAP之间相互隔离。
               4, Passthrough, 可以配合直接使用SRIOV网卡, 内核的macvtap数据处理逻辑被跳过，硬件决定数据如何处理，从而释放了Host CPU资源。MACVTAP Passthrough 概念与PCI Passthrough概念不同，PCI Passthrough针对的是任意PCI设备，不一定是网络设备，目的是让Guest OS直接使用Host上的 PCI 硬件以提高效率。MACVTAP Passthrough仅仅针对 MACVTAP网络设备，目的是饶过内核里MACVTAP的部分软件处理过程，转而交给硬件处理。综上所述，对于一个 SRIOV 网络设备，可以用两种模式使用它：MACVTAP Passthrough 与 PCI Passthrough
         PCI-pass-through: 直通,设备独享
         SR-IOV: 优点是虚拟网卡的工作由host cpu交给了物理网卡来实现，降低了host cpu的使用率，缺点是，需要网卡、主板、hypervisor的支持。
    总结来看网络虚拟化具有三个层次:
        1, 0成本，通过纯软件virtio、vhost、macvtap提升网络性能;
        2, 也可以用非常低的成本按照802.1Qbg中的VEPA模型创建升级版的虚拟网络，引出虚拟机网络流量，减少Host cpu负载，但需要物理交换机的配合;
        3, 如果网络性能还是达不到要求，可以尝试SR-IOV技术，不过需要SR-IOV网卡的支持。

ref: ***kvm 性能优化方案: http://blog.csdn.net/beginning1126/article/details/41983547
      ***玩转CPU-TOPOLOGY: http://blog.itpub.net/645199/viewspace-1421876/

2. KVM 虚拟化CPU 技术总结:
ref:kvm 虚拟化cpu技术总结 http://www.tuicool.com/articles/7FVR32Y
3. cpu pinning
4. Cgroup
5. TODO Link:
http://blog.csdn.net/wyzxg/article/details/5661489
