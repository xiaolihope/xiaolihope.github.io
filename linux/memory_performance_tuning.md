1. The memory related parameters
1) EPT(Extended Page Table) and VPID (Virtual Processor Identifiers)
    vm vaddr -->vm padddr -->host paddr , EPT的引入将两次地址转换变成了一次. 是在BIOS中,随着VT技术的开启一起开启
    
2) Huge Page:
    普通大页:cpu默认使用的是4KB的内存页面. 但是他们也支持较大的内存页,比如2MB. (huge page)
            huge page --> 内存也数量减少 --> 更少的页表 -->  节约页表内存空间 --> 地址转换减少  --> TLB 缓存失效次数减少  --> 提高内存访问的性能.
             --> cpu 缓存压力减少 --> 提高系统性能
            优点:  对于内存访问密集型应用,在kvm客户机中使用huge page是可以较明显提高性能.
            缺点: 使用huge page的内存不能被swap out,也不能使用ballooning 方式自动增长.需要应用程序显示使用      
    1G大页: 
    transparent_hugepage :
        echo always > /sys/kernel/mm/transparent_hugepage/enabled  
        
3) memory overload use:
    3.1 swapping
        swap 是将磁盘上的一部分空间用作交换，称之为虚拟内存. 当swappiness 越低就是叫OS尽量使用物理内存，然后才是SWAP空间。数值越高就算叫OS尽量使用SWAP
                 默认是60，也就算说内存使用到100-60=40%的时候，就开始出现有交换分区的使用。
                 By setting it to the maximum of 100 the kernel will swap very aggressively. By setting it to 0 the kernel will only swap to protect against an out-of-memory condition. The default is 60 which means that some swapping will occur.
                 Swapping is bad in a virtual environment, at any level. When the guest OS starts swapping on a physical server it only affects that server, but in a virtual environment it causes problems for all the workloads.
                 How to stop swap?
                 1. $ sudo sysctl -w vm.swappiness=0
                 2.  appending “vm.swappiness = 0” to file /etc/sysctl.conf and sudo sysctl –p to reload the values.
                 3. change it temporary cat /proc/sys/vm/swappiness
                     60
                 link: https://lonesysadmin.net/2013/12/11/adjust-vm-swappiness-avoid-unneeded-disk-io/
                 Note: 最新的linux内核中，如果设置此值为0，将会产生内存溢出的问题. eg. 当内存溢出的时候，会kill掉消耗内存很大的cpu.
                 link: http://blog.csdn.net/longxibendi/article/details/38146521
            
                所以如果希望提高服务器的并发量，对服务的相应时间要求不很高的场景可以适当的把swappniess调节的高些。对于并发量不大但希望相应时间小的应用场景可以适当的调小这个参数，比如个人电脑可以直接禁掉swap。
    3.2 ballooning 
        内存的ballonning 技术可以使在客户机运行时动态地调整它所占用的宿主机的内存资源,而不需要关闭客户机. 
        优点: 节约内存和灵活分配内存; 能够被监控和控制
        缺点: 需要virtio_balloon 驱动; 如果有大量内存需要从客户机回收,会降低客户机的系统性能. 没有自动化的机制来管理  内存的动态增加和减少,可能会使被过度碎片化,内存策略会受影响.
    3.3 ksm(Kernel Samepage Merging)
         简单理解就是可以将host机内容相同的内存合并，节省内存的使用，特别是当vm操作系统都一样的情况，肯定会有很多内容相同的内存，开启了KSM，则会将这些内存合并为一个，当然这个过程会有性能损耗，平均性能降低为10%， 所以开启与否，需要考虑使用场景，如果不注重vm性能，而注重host内存使用率，可以考虑开启，反之 则关闭，在/etc/init.d/下，会有两个服务，服务名称为ksmd和ksmtuned，都需要关闭。
          (rhel6 and fedora 14)默认是开启的.
           ref link:KSM: http://searchenterpriselinux.techtarget.com/tip/How-to-improve-KVM-performance-by-adjusting-KSM?utm_content=Linuxclusteract
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
4. 内存碎片整理开启
    echo always> /sys/kernel/mm/transparent_hugepage/defrag
5. cache policy  
    vm.dirty_background_ration
    vm.dirty.ration
         dirty: 是指需要写回磁盘的数据大小
vm.dirty_background_ration: 这个参数指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如5%）就会触发pdflush/flush/kdmflush等后台回写进程运行，将一定缓存的脏页异步地刷入外存；
vm.dirty_ratio: 而这个参数则指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如10%），系统不得不开始处理缓存脏页（因为此时脏页数量已经比较多，为了避免数据丢失需要将一定脏页刷入外存）；在此过程中很多应用进程可能会因为系统转而处理文件IO而阻塞。
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
              test: http://labs.isee.biz/index.php/Board_validation_and_diagnostic_tools
6. Memory tuning
ref link: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-libvirt-dom-xml-mem-tuning.html
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
