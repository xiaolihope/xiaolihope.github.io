---
title: Virsh
date: 2019-08-26
---

## virsh 常用命令

```markdown
# virsh --help                                     #查看命令帮忙
# virsh list                                       #显示正在运行的虚拟机
# virsh list --all                                 #显示所有的虚拟机

# virsh start vm-node1                             #启动vm-node1虚拟机
# virsh shutdown vm-node1                          #关闭vm-node1虚拟机
# virsh destroy vm-node1                           #虚拟机vm-node1强制断电
# virsh suspend vm-node1                           #挂起vm-node1虚拟机
# virsh resume vm-node1                            #恢复挂起的虚拟机

# virsh define <libvrit.xml>
# virsh undefine vm-node1                          #删除虚拟机，慎用

# virsh dominfo vm-node1                           #查看虚拟机的配置信息
# virsh domiflist vm-node1                         #查看网卡配置信息
# virsh domblklist vm-node1                        #查看该虚拟机的磁盘位置
# virsh vncdisplay vm-node1

# virsh edit vm-node1                              #修改vm-node1的xml配置文件
# virsh dumpxml vm-node1                           #查看KVM虚拟机当前配置
# virsh dumpxml vm-node1 > vm-node1.bak.xml        #备份vm-node1虚拟机的xml文件，原文件默认路径/etc/libvirt/qemu/vm-node1.xml

# virsh autostart vm-node1                         #KVM物理机开机自启动虚拟机，配置后会在此目录生成配置文件/etc/libvirt/qemu/autostart/vm-node1.xml
# virsh autostart --disable vm-node1
```
## virsh console 登陆虚拟机
有两种方法可以登陆virsh创建的虚拟机，
第一种：vnc, 用virsh vncdisplay <vm_name>,得到:0，然后用vnc viewer，输入宿主机ip：0,进入到虚拟机；
第二种：virsh console

第一步：添加ttyS0的许可，允许root登陆
```markdown
# echo "ttyS0" >> /etc/securetty
```
第二步：在宿主机上测试连接
```markdown
[root@study ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 4     centos                         running
 
[root@study ~]# virsh console 4
Connected to domain centos
Escape character is ^]
 
CentOS release 6.5 (Final)
Kernel 2.6.32-431.el6.x86_64 on an x86_64
 
localhost.localdomain login: root
Password: 
Last login: Thu Oct 13 02:51:30 on ttyS0
[root@localhost ~]#
注：mac上按 ctrl+] 组合键退出virsh console
```

## virsh 为虚拟机增加网卡
一个完整的数据包从虚拟机到物理机的路径是：虚拟机-->QEMU虚拟机网卡-->虚拟化层-->内核网桥-->物理网卡

KVM默认情况下是由QEMU在Linux的用户空间模拟出来的并提供给虚拟机的。

全虚拟化：即客户机操作系统完全不需要修改就能运行于虚拟机中，客户机看不到真正的硬件设备，与设备的交互全是由纯软件虚拟的

半虚拟化：通过对客户机操作系统进行修改，使其意识到自己运行在虚拟机中。因此，全虚拟化和半虚拟化网卡的区别在于客户机是否需要修改才能运行在宿主机中。

半虚拟化使用virtio技术，virtio驱动因为改造了虚拟机的操作系统，让虚拟机可以直接和虚拟化层通信，从而大大提高了虚拟机性能。

```markdown
[root@kvm-server ~]# virsh domiflist vm-node1
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     br0        virtio      52:54:00:40:75:05

[root@kvm-server ~]# virsh attach-interface vm-node1 --type bridge --source br0 --model virtio        #临时增加网卡的方法，关机后再开机新增网卡配置丢失
Interface attached successfully

[root@kvm-server ~]# virsh domiflist vm-node1
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     br0        virtio      52:54:00:40:75:05
vnet1      bridge     br0        virtio      52:54:00:5b:6c:cc

[root@kvm-server ~]# virsh edit vm-node1                                                               #永久生效方法一：修改配置文件增加如下内容
    <interface type='bridge'>                                                                          #永久生效方法二：使用virt-manager管理工具进行操作
      <mac address='52:54:00:11:90:7c'/>
      <source bridge='br0'/>
      <target dev='vnet1'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </interface>

[root@kvm-server ~]# virsh domiflist vm-node1                                                          #查找虚拟机网卡的MAC地址
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     br0        virtio      52:54:00:40:75:05
vnet1      bridge     br0        virtio      52:54:00:84:23:3d

[root@kvm-server ~]# virsh detach-interface vm-node1 --type bridge --mac 52:54:00:84:23:3d --current   #根据MAC地址删除网卡，即时生效，如果需要最终生效也要使用virsh edit 来修改配置文件
Interface detached successfully

[root@kvm-server ~]# virsh domiflist vm-node1
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     br0        virtio      52:54:00:40:75:05
```

## virsh 增加磁盘

KVM虚拟机的磁盘镜像从存储方式上看，可以分为两种方式，第一种方式为存储于文件系统上，第二种方式为直接使用裸设备。裸设备的使用方式可以是直接使用裸盘，也可以是用LVM的方式。存于文件系统上的镜像有很多格式，如raw、cloop、cow、qcow、qcow2、vmdlk、vdi等，经常使用的是raw和qcow2。

raw：是简单的二进制镜像文件，一次性会把分配的磁盘空间占用。raw支持稀疏文件特性，稀疏文件特性就是文件系统会把分配的空字节文件记录在元数据中，而不会实际占用磁盘空间。

qcow2：第二代的QEMU写时复制格式，支持很多特性，如快照、在不支持稀疏特性的文件系统上也支持精简方式、AES加密、zlib压缩、后备方式。

```markdown
[root@kvm-server ~]# qemu-img create -f raw /Data/vm-node1-10G.raw 10G                                #创建raw格式并且大小为10G的磁盘
Formatting '/Data/vm-node1-10G.raw', fmt=raw size=10737418240 

[root@kvm-server ~]# qemu-img info /Data/vm-node1-10G.raw 
image: /Data/vm-node1-10G.raw
file format: raw
virtual size: 10G (10737418240 bytes)
disk size: 0

[root@kvm-server ~]# virsh attach-disk vm-node1 /Data/vm-node1-10G.raw vdb --cache none                #临时生效，关机再开机后失效
Disk attached successfully

[root@kvm-server ~]# virsh dumpxml vm-node1                                                            #通过dumpxml找到下段配置文件
[root@kvm-server ~]# virsh edit vm-node1                                                               #使用edit命令，把找到的内容加到vda磁盘后面即可
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' cache='none'/>
      <source file='/Data/vm-node1-10G.raw'/>
      <backingStore/>
      <target dev='vdb' bus='virtio'/>
      <alias name='virtio-disk1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
    </disk>

[root@vm-node1 ~]# fdisk -l                                                                            #数据盘已挂载，可以进行分区、格式化、挂载等操作

Disk /dev/vda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00009df9

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    83886079    41942016   83  Linux

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

磁盘镜像格式的转换方法：

[root@kvm-server ~]# qemu-img create -f raw test.raw 5G
Formatting 'test.raw', fmt=raw size=5368709120
[root@kvm-server ~]# qemu-img convert -p -f raw -O qcow2 test.raw test.qcow2                              #参数-p显示进度，-f是指原有的镜像格式，-O是输出的镜像格式，然后是输入文件和输出文件
    (100.00/100%)
[root@kvm-server ~]# qemu-img info test.qcow2 
image: test.qcow2
file format: qcow2
virtual size: 5.0G (5368709120 bytes)
disk size: 196K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false

[root@kvm-server ~]# ll -sh test.*                                        
196K -rw-r--r-- 1 root root 193K Oct 19 16:19 test.qcow2
-rw-r--r-- 1 root root 5.0G Oct 19 16:11 test.raw
```

## 克隆虚拟机

使用virt-clone克隆虚拟机的方法：
```markdown
[root@kvm-server ~]# virsh shutdown CentOS-7.2-x86_64                                                      #必须要关机才能进行克隆
Domain CentOS-7.2-x86_64 is being shutdown

[root@kvm-server ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 -     CentOS-7.2-x86_64              shut off
 -     vm-node1                       shut off

[root@kvm-server ~]# virt-clone -o CentOS-7.2-x86_64 -n vm-node2 -f /opt/vm-node2.raw                       #参数含义：-o被克隆虚拟机的名字、-n克隆后虚拟机的名字、-f指定磁盘存储位置
WARNING  The requested volume capacity will exceed the available pool space when the volume is fully allocated. (40960 M requested capacity > 36403 M available)
Allocating 'vm-node2.raw'                                                                                                             |  40 GB  00:01:03     

Clone 'vm-node2' created successfully.
[root@kvm-server ~]# virsh list --all                                                                        #克隆后为关机状态
 Id    Name                           State
----------------------------------------------------
 -     CentOS-7.2-x86_64              shut off
 -     vm-node1                       shut off
 -     vm-node2                       shut off
```

## 修改虚拟机名字

```markdown
[root@kvm-server ~]# virsh shutdown CentOS-7.2-x86_64                                                         #需要先关机，然后对虚拟机进行改名
[root@kvm-server ~]# cp /etc/libvirt/qemu/vm-node2.xml /etc/libvirt/qemu/vm-test.xml                          #拷贝xml文件为要修改的名称，如:vm-test
[root@kvm-server ~]# grep '<name>' /etc/libvirt/qemu/vm-test.xml                                              #修改vm-test.xml中的name字段为vm-test
  <name>vm-test</name>
[root@kvm-server ~]# virsh undefine vm-node2                                                                  #删除之前的虚拟机
Domain vm-node2 has been undefined

[root@kvm-server ~]# virsh define /etc/libvirt/qemu/vm-test.xml                                               #定义新的虚拟机
Domain vm-test defined from /etc/libvirt/qemu/vm-test.xml

[root@kvm-server ~]# virsh list --all                                                                         #已完成改名操作
 Id    Name                           State
----------------------------------------------------
 -     CentOS-7.2-x86_64              shut off
 -     vm-node1                       shut off
 -     vm-test                        shut off
```