---
date: 2016-09-10T16:06:04+08:00
title: getting started with heat
---

## Heat Introduction

Heat openstack docs: [heat-doc]

![heat-architecture](../images/heat-architecture.png)

## Heat Architecture

- Heat-api: 实现openstack天然支持的REST API. 该组件通过把API请求经由AMQP传送给 Heat engine 来处理 API 请求 .
- Heat-api-cfn:提供兼容AWSCloudFormation的API,同时也会把API 请求通过AMQP转发 给 Heat engine
- Heat-engine: 提供Heat主要的功能
- heat-api-cloud-watch: 负责资源监控的,现在已经不再用了

![heat-components](../images/heat-components.png)

Heat 主要分为 heat client,heat api,heat engine. 调用逻辑如下 :

Heat client 接受输入命令 , 参数 , 和模板 (URL, 文件路径or 数据),处理信息后转为REST API

请求发送到heat-api 服务.

Heat API 服务接受请求 , 读入模板信息 , 处理后 利用rpc请求发送给 heat-engine.

heat-engine 解析template数据,调用各种资源插件 , 然后各种资源插件通过 openstack 的 clients 发送指 令给 openstack 服务 .

[heat architecture]
### Use Heat

Write heat templates and create one stack:

```
heat_template_version: 2013-05-23

description: >
  HOT template to create a new neutron network plus a router to the public
  network, and for deploying three servers into the new network. The template also
  assigns floating IP addresses to each server so they are routable from the
  public network.

parameters:
  key_name:
    type: string
    description: Name of keypair to assign to servers
    default: dxl-key
  image:
    type: string
    description: Name of image to use for servers
    default: cirros-0.3.5
  flavor:
    type: string
    description: Flavor to use for servers
    default: m1.tiny
  public_net:
    type: string
    description: >
      ID or name of public network for which floating IP addresses will be allocated
    default: public
  private_net_name:
    type: string
    description: Name of private network to be created
    default: private_test_net
  private_net_cidr:
    type: string
    description: Private network address (CIDR notation)
    default: 182.168.1.1/24
  private_subnet_name:
    type: string
    description: Private subnet name
    default: private_test_subnet

resources:
  private_net:
    type: OS::Neutron::Net
    properties:
      name: { get_param: private_net_name }
  private_subnet:
    type: OS::Neutron::Subnet
    properties:
      name: {get_param: private_subnet_name}
      network_id: { get_resource: private_net }
      cidr: { get_param: private_net_cidr }
  router:
    type: OS::Neutron::Router
    properties:
      external_gateway_info:
        network: { get_param: public_net }
  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router_id: { get_resource: router }
      subnet_id: { get_resource: private_subnet }

  servers:
    type: OS::Heat::ResourceGroup
    properties:
      count: 1
      resource_def:
        type: OS::Nova::Server
        properties:
          image: { get_param: image }
          flavor: { get_param: flavor }
          key_name: { get_param: key_name }
          networks:
            - network: { get_resource: private_net }
```

![heat-dashborard](../images/heat-dashboard.png)


## Heat Glossary

### Stack
![heat-stack](../images/heat-stack.png)

### Templates
[heat template guide]

## Heat Main Function
### 1. Heat Resource Type

[heat resource type]

## Heat Software Configuration

[heat software configuration]

### 2. AutoScalling

## Stack LifeCycle

整个 heat 就是围绕 stack 在玩 , 管理 stack 的生命周期.

ACTION +  STATUS

ACTIONS: 'CREATE', 'DELETE', 'UPDATE', 'ROLLBACK', 'SUSPEND', 'RESUME', 'ADOPT', 'SNAPSHOT', 'CHECK', 'RESTORE'
status: 'IN_PROGRESS', 'FAILED', 'COMPLETE'

## stack domain users
https://docs.openstack.org/admin-guide/orchestration-stack-domain-users.html

[heat-doc]: https://docs.openstack.org/heat/latest/
[heat template guide]: https://docs.openstack.org/heat/latest/template_guide/index.html
[heat architecture]: https://docs.openstack.org/heat/latest/developing_guides/architecture.html
[heat resource type]: https://docs.openstack.org/heat/latest/template_guide/openstack.html
[heat software configuration]: https://docs.openstack.org/heat/latest/template_guide/software_deployment.html
