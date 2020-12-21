---
date: 2016-12-20T01:13:50+08:00
title: ceph
---

## Configure cinder volume with ceph backend

### create a pool

```
    # ceph osd pool create volumes-dxl 256
    pool 'volumes-dxl' created
    # ceph osd pool get volumes-dxl pgp_num
    pgp_num: 256
```

### on the openstack environment (controller node)

step1:  install ceph client packages

    # sudo apt-get install ceph-common -y
    # sudo apt-get install python-rbd -y
    
step2: configure openstack ceph.conf files
        
    copy the ceph.conf from t1 env to openstack env.
        
step3: setup ceph client authentication on t1

        # ceph auth get-or-create client.cinder-dxl mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes-dxl'
        [client.cinder-dxl]
                key = AQBXoMZXBeiCCBAA5Dd03uwMux5cWOiAobphBA==
        root@t1-api-02:/etc/ceph# ceph auth get-or-create client.cinder-dxl | tee /etc/ceph/ceph.client.cinder-dxl.keyring
        [client.cinder-dxl]
                key = AQBXoMZXBeiCCBAA5Dd03uwMux5cWOiAobphBA==
                
step4: Add the keyrings for cinder-dxl  to cinder-volume nodes and change the owership.

        # cat >> ceph.client.cinder-dxl.keyring
        [client.cinder-dxl]
                key = AQBXoMZXBeiCCBAA5Dd03uwMux5cWOiAobphBA==
        and change the owership
        # chown cinder:cinder ceph.client.cinder-dxl.keyring
        
step5: also copied the admin keying to the compute nodes in order to make ceph command works.

### configure openstack cinder to use ceph on the api nodes

```
    # cat /etc/cinder/cinder.conf
    [DEFAULT]
    enabled_backends = ceph
    
    [ceph]
    volume_driver = cinder.volume.drivers.rbd.RBDDriver
    rbd_pool = volumes-dxl
    rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_flatten_volume_from_snapshot = false
    rbd_max_clone_depth = 5
    rbd_store_chunk_size = 4
    rados_connect_timeout = -1
    glance_api_version = 2
    rbd_user = cinder-dxl
    rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

# restart cinder-volume service
# cinder service-list

```

### configure the compute nodes

```                               
    step1: install ceph client on all the compute nodes
    #sudo apt-get install ceph-common -y
    step2: copy the ceph.conf and admin/cinder-dxl keying to the compute nodes.
    
    step3: set the libvirt-secret.xml
    # cat libvirt-secret.xml
    <secret ephemeral='no' private='no'>
      <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
      <usage type='ceph'>
        <name>AQBXoMZXBeiCCBAA5Dd03uwMux5cWOiAobphBA==</name>
      </usage>
    </secret>
    
    # virsh secret-define --file /etc/ceph/libvirt-secret.xml
    Secret 457eb676-33da-42ec-9a8c-9293d545c337 created
    
    # virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 --base64 "AQBXoMZXBeiCCBAA5Dd03uwMux5cWOiAobphBA=="
    Secret value set
    step4:  configure nova-compute.conf
    rbd_user = cinder-dxl
    rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
    
    # restart nova-compute service

```
ref link http://docs.ceph.com/docs/master/rbd/rbd-openstack/
and https://w3-connections.ibm.com/wikis/home?lang=en-us#!/wiki/W2e52de5e48f0_4f00_a00f_5ce1f8e43d09/page/Setup%20Ceph%20Cluster%20in%20Honor

## ceph commands

```
# ceph -s //查看信息状态
# ceph -df 
# ceph auth list
# ceph auth del client.cinder_dxl
# rados lspools // list pools
# rados df  // view the detailed messgae of the pools
# rados mkpool test // create a pool 
# ceph osd pool delete  dxl_images dxl_images --yes-i-really-really-mean-it // delete  a pool
# ceph osd pool get <pool-name> pg_num
# rbd -p pool_name ls // list the volumes
# rbd -p pool_name remove volume_name 

```