# Boot Vm From Iso Image
2018/03/13

本文介绍nova从iso image创建vm的方式以及对此进行分析。

## 问题

在kilo/pike版本上，用iso格式的 image 创建vm（从卷启动），vm 启动时，报错 `No bootable device`

```
# nova boot  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=image,id=0addb309-7634-4c7d-b0ee-8e8bf0b4c913,dest=volume,size=30,shutdown=preserve,bootindex=0 test
# nova boot  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=volume,id=211e31f4-3cf7-4a5b-b882-abe004ca0371,dest=volume,size=30,shutdown=preserve,bootindex=0 test
```

这两种方式，vm 的系统盘都是用cinder 创建的卷。
## 分析
### 查看vm的libvirt.xml
```
<boot dev="hd"/>
<disk type="network" device="disk">
      <driver name="qemu" type="raw" cache="none"/>
      <source protocol="sheepdog" name="volume-76b13500-100a-4310-807c-7997a2544701">
        <host name="127.0.0.1" port="7050"/>
      </source>
      <target bus="virtio" dev="vda"/>
      <serial>76b13500-100a-4310-807c-7997a2544701</serial>
    </disk>
```
用
```
# nova boot --image <image-Id> --flavor <flavor_id> --nic net-id=<network_id> <instance_name>
```
or
```
# nova boot --image 0addb309-7634-4c7d-b0ee-8e8bf0b4c913  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=blank,dest=volume,size=30,shutdown=preserve test
```
这种方式部署的vm, 启动成功。查看libvirt.xml 文件如下:
```
<os>
  <type>hvm</type>
  <boot dev="cdrom"/>
  <smbios mode="sysinfo"/>
</os>
<disk type="file" device="cdrom">
  <driver name="qemu" type="qcow2" cache="none"/>
  <source file="/var/lib/nova/instances/b7bccd6b-e283-4763-b577-d200208a235e/disk"/>
  <target bus="ide" dev="hda"/>
</disk>
```
通过将上面的disk下载到本地，然后转换成qcow2, 再将disk bus修改成virtio之后可以启动成功。
```
qemu-img convert -f raw -O raw sheepdog:127.0.0.1:7050:volume-<volume_id> <disk_name>
```
由此可见diskbus=ide,disk_type=qcow2 是boot iso image vm 启动成功的原因。

##  总结
### nova boot 的几种方式
1. boot from image
```
# nova boot --image --flavor --nic net-id=
```

2. boot with block_device
```
# nova boot  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=volume,id=211e31f4-3cf7-4a5b-b882-abe004ca0371,dest=volume,size=30,shutdown=preserve,bootindex=0 test 问题重现  recreate with —image, and delete bootindex=0,it can works well.
# nova boot  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=image,id=0addb309-7634-4c7d-b0ee-8e8bf0b4c913,dest=volume,size=30,shutdown=preserve,bootindex=0 test 问题重现 recreate with —image, and delete bootindex=0,it can works well.
# nova boot --image 0addb309-7634-4c7d-b0ee-8e8bf0b4c913  --flavor 2 --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2 --block-device source=blank,dest=volume,size=30,shutdown=preserve test // will create a volume named “45ef5985-f270-44e0-874b-dc8bf66d83af-blank-vol”
# nova boot --flavor 1 --block-device-mapping vda=7b5a4b7a-2bea-4b00-af0c-2f68d6666597:::0 test --nic net-id=eedd8d5f-0728-4d2c-afd1-4b845b12b0c2
```
3. boot from lvm volume
```
<disk type="block" device="disk">
  <driver name="qemu" type="raw" cache="none"/>
  <source dev='/dev/sdm'/>
  <target bus="ide" dev="hda"/> // ide 和 virtio 都不work
  <serial>15cc802e-a88b-4e30-910b-ff71cebb4b20</serial>
</disk>
```

### 从iso部署vm，官方文档
- [Juno]
```
nova boot --image ubuntu.iso --flavor 1 instance_name
```  
- [kilo]
```  
nova boot \
--image ubuntu-14.04.2-server-amd64.iso \
--block-device source=blank,dest=volume,size=10,shutdown=preserve \
--nic net-id=NETWORK_UUID
--flavor 3 INSTANCE_NAME
```
- [liberty]
```
nova boot --image ubuntu.iso  --flavor 1 instance_name
```
- [mitaka]
```  
nova boot \
--image ubuntu-14.04.2-server-amd64.iso \
--block-device source=blank,dest=volume,size=10,shutdown=preserve \
--nic net-id = NETWORK_UUID \
--flavor 2 INSTANCE_NAME
```
- [newton]
```
nova boot \
--image ubuntu-14.04.2-server-amd64.iso \
--block-device source=blank,dest=volume,size=10,shutdown=preserve \
--nic net-id = NETWORK_UUID \
--flavor 2 INSTANCE_NAME
  ```
- [ocata] 
  
boot from image

- [pike]
```
openstack server create --image ubuntu-14.04.2-server-amd64.iso \
--nic net-id = NETWORK_UUID \
--flavor 2 INSTANCE_NAME
```

[Juno]: https://docs.openstack.org/juno/config-reference/content/iso-support.html
[kilo]: http://docs.openstack.org/kilo/config-reference/content/iso-support.html
[liberty]: https://docs.openstack.org/liberty/config-reference/content/iso-support.html
[mitaka]: https://docs.openstack.org/mitaka/user-guide/cli_nova_launch_instance_using_ISO_image.html
[newton]: https://docs.openstack.org/newton/user-guide/cli-nova-launch-instance-using-ISO-image.html
[ocata]: https://docs.openstack.org/ocata/user-guide/cli-nova-launch-instance-using-ISO-image.html
[pike]: https://docs.openstack.org/nova/pike/user/launch-instance-using-ISO-image.html#boot-an-instance-from-an-iso-image