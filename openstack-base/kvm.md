---
date: 2016-09-15T14:20:45+08:00
title: KVM
---

1. KVM 是什么？
KVM (全称是 Kernel-based Virtual Machine) 是 Linux 下 x86 硬件平台上的全功能虚拟化解决方案，包含一个可加载的内核模块 kvm.ko 提供和虚拟化核心架构和处理器规范模块。 使用 KVM 可允许多个包括 Linux 和 Windows 每个虚拟机有私有的硬件，包括网卡、磁盘以及图形适配卡等。
2. KVM qemu-kvm libvirt virsh
link:
1. http://my.oschina.net/qefarmer/blog/386843
2. https://libvirt.org/drvqemu.html
3. http://www.tuicool.com/articles/i63Ivy
4. http://www.xuebuyuan.com/2068851.html
5. http://qemu-buch.de/
6. 概念之间的关系:  http://www.2cto.com/os/201305/209596.html

virt-manager

link:
1. kvm 虚拟化技术之Hypervisor 架构 http://www.oschina.net/question/2548918_2149938
2. kvm 虚拟化技术之Hypervisor 实现 http://www.oschina.net/question/2548918_2150436


chapter 2:  KVM 原理简介
虚拟化模型
客户机虚拟机1; 虚拟机2;虚拟机3
      |
VMM 虚拟机监控器 or Hypervisor: 职责: 管理真实的物理硬件平台,并为每个虚拟客户机提供对应的虚拟硬件平台. 
      |
底层硬件

KVM 架构
(半虚拟化)类型一:系统上电之后,首先加载运行虚拟机监控程序  Xen, VMware ESX/ESXi,  Hyper-V 
(全虚拟化)类型二: 系统上电之后,首先加载运行宿主机操作系统  KVM, VMware Workstation, VirtualBox 
KVM 模块
CPU 虚拟化 kvm 加载--> /dev/kvm --> QEMU and KVM 配合对/dev/kvm 进行IOCTL 调用(也就是创建虚拟机) 
内存虚拟化 
Intel 虚拟化技术
三类:
1) 与处理器相关的 VT-x
2) 芯片组相关的 VT-d
3) 输入输出设备相关的: 主要目的是通过定义新的输入输出协议,使新一代的输入输出设备可以更好的支持虚拟化环境下的工作. 
    VMDq and  PCI 组织定义的单根设备虚拟化协议 (SR-IOV)
Chapter3: 构建KVM环境
1. 硬件系统的配置
1) CPU enable VT 
    setting in BIOS: Advanced -> Processor Configuration -> VT
    if support VT-d: 
        Intel(R) VT for Directed I/O or Intel VT-d 
check: /proc/cpuinfo flags
$ grep -E '(vmx|svm)' /proc/cpuinfo
2. 宿主机(KVM)操作系统的安装
kvm是基于内核的虚拟化技术,要运行KVM虚拟化环境,安装一个Linux操作系统的宿主机(Host)是必需的. 
后面会使用自己编译的kernel and qemu-kvm 来进行实验.
[optional]3. KVM的编译与安装
1) 下载最新的kvm 源代码
[optional]4. qemu-kvm的编译与安装
command: 
qemu-system-x86_64 
qemu-img 
5. 客户机(Guest)的安装
6. 启动客户机

Chapter 4: KVM核心基础功能
1. CPU: 
    processor affinity 进程的处理器亲和性 即CPU的绑定设置,是指将进程绑定到待定的一个或者多个CPU上去执行,而不允许将进程调度到其它CPU上.
    cpu isolation: page 58.
2. memory
    $free -m //查看内存信息,显示的总内存是除了内核执行文件占用内存和一些系统保留的内存之后能使用的内存.
    $dmesg  
    $/proc/meminfo
    1) EPT: 
    2) 大页: cpu默认使用的是4KB的内存页面. 
            但是他们也支持较大的内存页,比如2MB. (huge page)
            huge page --> 内存也数量减少 --> 更少的页表 -->  节约页表内存空间 --> 地址转换减少  --> TLB 缓存失效次数减少  --> 提高内存访问的性能.
             --> cpu 缓存压力减少 --> 提高系统性能
            优点:  对于内存访问密集型应用,在kvm客户机中使用huge page是可以较明显提高性能.
            缺点: 使用huge page的内存不能被swap out,也不能使用ballooning 方式自动增长.需要应用程序显示使用
        1G大页
        透明大页: 大页--> 优化-->透明大页  
             可以交换
    3) 内存过载使用:
        内存交换:swapping
        气球: ballooning 
            内存的ballonning 技术可以使在客户机运行时动态地调整它所占用的宿主机的内存资源,而不需要关闭客户机. 
            优点: 节约内存和灵活分配内存
                    能够被监控和控制
            缺点: 需要virtio_balloon 驱动
                    如果有大量内存需要从客户机回收,会降低客户机的系统性能.
                    没有自动化的机制来管理
                    内存的动态增加和减少,可能会使被过度碎片化,内存策略会受影响.
        页共享: ksm





