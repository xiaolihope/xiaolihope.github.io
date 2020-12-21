---
title: "OVS"
date: 2019-08-27
---

## ovs-vsctl commands


```markdown
# ovs-vsctl add-br <bridge_name>                 // 创建网桥
# ovs-vsctl add-port <bridge_name> <nic_name>    // 网桥添加端口

# ovs-vsctl set bridge br-eth1 rstp_enable=true  // 开启网桥rstp协议
# ovs-vsctl get bridge br-eth1 rstp_enable       // 查看网桥rstp开启状态


# ovs-vsctl set port <port_name> tag=10         // ovs设置端口vlan tag

// patch连接两个网桥
# ovs-vsctl add-port <bridge-name> <port-name>
# ovs-vsctl set interface <port-name> type=patch
# ovs-vsctl set interface <port-name> option:peer=<peer-name>
```

## ovs-ofctl commands
```markdown
# ovs-ofctl show br-eth1                           // 查看网桥详细信息
```

