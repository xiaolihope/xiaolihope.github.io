# Libvirt create VM
date: 2018-03-13T11:40:36+08:00

本文介绍用libvirt 启动vm

- 方法一：`virsh define`

创建一个空镜像

```
# qemu-img create -f qcow2 test.qcow2 100G
```
定义 libvirt.xml 如下：
```
<domain type="kvm">
  <name>instance-dxl-fedora</name>
  <memory>4194304</memory>
  <vcpu cpuset="12-23,36-47">2</vcpu>
  <os>
    <type>hvm</type>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <clock offset="utc">
    <timer name="pit" tickpolicy="delay"/>
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="hpet" present="no"/>
  </clock>
  <cpu mode="host-passthrough" match="exact">
    <topology sockets="2" cores="1" threads="1"/>
  </cpu>
  <devices>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/var/log/dixiaoli/Fedora-Server-dvd-x86_64-27-1.6.qcow2">
      </source>
      <target bus="virtio" dev="hda"/>
    </disk>
    <disk device="cdrom" type="file">
      <target bus="ide" dev="hdc"/>
      <source file="/var/log/dixiaoli/Fedora-Server-dvd-x86_64-27-1.6.iso"/>
      <driver type="raw" name="qemu"/>
    </disk>
    <interface type="bridge">
      <model type="virtio"/>
      <source bridge="qbrdb858da5-68"/>
    </interface>
    <input type="tablet" bus="usb"/>
    <graphics type="vnc" autoport="yes" keymap="en-us" listen="10.160.58.23"/>
    <video>
      <model type="cirrus"/>
    </video>
    <memballoon model="virtio">
      <stats period="10"/>
    </memballoon>
  </devices>
</domain>
```
`<domain type="qemu">`适用于vm的情况。

```
# virsh define libvirt.xml
# virsh start <domain_name>
# virsh vncdisplay <domain_name>
# virsh destroy <domain_name>
```

Note: 修改了libvirt.xml文件之后，需要`virsh undefine <domain_name>`  再重新`define`.

- 方法二：`virsh-install`

本方法参考openstack提供的创建image的文档:[Create Image Manually](https://docs.openstack.org/image-guide/create-images-manually.html)

```
# virsh net-list
```
if there is none, you should try: yum install libvirt-daemon-config-network -y
then
virsh net-start default
```
# qemu-img create -f qcow2 /tmp/centos.qcow2 10G

Formatting '/tmp/centos.qcow2', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16
# yum install -y virt-install
# virt-install --virt-type qemu --name centos --ram 1024 --disk /tmp/centos.qcow2,format=qcow2,bus=virtio --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole --os-type=linux --os-variant=centos7.0 --location=/tmp/TusCloudOS-latest.iso
Starting install...
Retrieving file .treeinfo...                                                                                                                                           |  354 B  00:00:00
Retrieving file vmlinuz...                                                                                                                                             | 5.6 MB  00:00:00
Retrieving file initrd.img...                                                                                                                                          |  46 MB  00:00:00
Domain installation still in progress. You can reconnect to
the console to complete the installation process.

or disk=none
```