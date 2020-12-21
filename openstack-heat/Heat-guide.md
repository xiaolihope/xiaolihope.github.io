*date: 2017-04-24T14:51:14+08:00*

## Heat 历史

由AWS的Cloud Formation 演化而来，是openstack中负责Orchestration的service。

## Heat Install

### Heat手动部署

### Devstack Install Heat
```
step1: git clone devstack
step2: tool/create-stack-user
step3: su stack
step4: prepare local.conf
```

```
cat local.conf

[[local|localrc]]
ADMIN_PASSWORD=admin
MYSQL_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD
GIT_BASE=https://git.openstack.org

SERVICE_TIMEOUT=180
API_WORKERS=1

REGION_NAME=RegionOne
KEYSTONE_TOKEN_FORMAT=UUID

OS_BRANCH=master

# Neutron services
disable_service n-net
enable_service q-svc q-agt q-dhcp q-l3 q-meta neutron
enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas
enable_service q-lbaasv2
NEUTRON_LBAAS_SERVICE_PROVIDERV2='LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default'
disable_service q-lbaasv1 q-lbaas
LIBS_FROM_GIT+=',python-neutronclient'

# Tempest
disable_service tempest

# Heat services
HEAT_ENABLE_ADOPT_ABANDON=True
enable_plugin heat https://git.openstack.org/openstack/heat
LIBS_FROM_GIT+=',python-heatclient'
IMAGE_URL_SITE="http://download.fedoraproject.org"
IMAGE_URL_PATH="/pub/fedora/linux/releases/25/CloudImages/x86_64/images/"
IMAGE_URL_FILE="Fedora-Cloud-Base-25-1.3.x86_64.qcow2"
IMAGE_URLS+=","$IMAGE_URL_SITE$IMAGE_URL_PATH$IMAGE_URL_FILE


# Ceilometer services
#CEILOMETER_BACKEND=mongodb
#CEILOMETER_PIPELINE_INTERVAL=600
#enable_plugin ceilometer https://git.openstack.org/openstack/ceilometer $OS_BRANCH

# Enable the aodh alarming services
#enable_plugin aodh https://git.openstack.org/openstack/aodh $OS_BRANCH

# Enable gnocchi services
#enable_plugin gnocchi https://git.openstack.org/openstack/gnocchi $OS_BRANCH

# Sahara services
#enable_plugin sahara https://git.openstack.org/openstack/sahara $OS_BRANCH

# Zaqar service
#enable_plugin zaqar https://git.openstack.org/openstack/zaqar $OS_BRANCH

# Senlin service
#enable_plugin senlin https://git.openstack.org/openstack/senlin $OS_BRANCH
#LIBS_FROM_GIT+=',python-senlinclient'

# Magnum service
#enable_plugin magnum https://git.openstack.org/openstack/magnum $OS_BRANCH

# Murano service
enable_plugin murano https://git.openstack.org/openstack/murano $OS_BRANCH

# Barbican service
#enable_plugin barbican https://git.openstack.org/openstack/barbican

[[post-config|$NOVA_CONF]]
[DEFAULT]
osapi_compute_workers=1
ec2_workers=1
metadata_workers=1
[conductor]
workers=1

[[post-config|$NEUTRON_CONF]]
[DEFAULT]
api_workers=1

[[post-config|$HEAT_CONF]]
[DEFAULT]
convergence_engine=true
notification_driver=messagingv2
num_engine_workers=1
hidden_stack_tags=hidden
encrypt_parameters_and_properties=True
[heat_api]
workers=1
[heat_api_cfn]
workers=1
[heat_api_cloudwatch]
workers=1
[cache]
enabled=True

```
step5: Run ./stack.sh

heat.conf 文件解析：
...

## Heat Architecture

heat 由以下组件构成：

```
python-heatclient
heat-api
heat-api-cfn
heat-engine
heat-cfntools
```
### heat template and stack

### nested stack

## Heat 功能

heat提供了如下功能：

### heat stack-create
```
-f  指定模版文件
-e 指定environment 文件，可以指定多次
--pre-create
-r 允许在 创建／更新 失败的时候回滚， 创建回滚就是会删除失败的stack，就好像没有创建过一样；更新回滚就是回到更新之前的状态
```
heat stack 创建过程分析：

### heat stack-update

## heat template

heat 提供了比较全的基本的template 库: https://github.com/openstack/heat-templates/tree/master/hot 当我们要写template的时候，
往往可以将此作为参考来写自己的template，会节省很多时间。

[template guide]

### heat_template_version

有两种写法：

- heat_template_version: 2013-05-23

- heat_template_version: mitaka

template version决定了，template中使用的resource type 和其支持的函数。

### description
### parameters
parameters的类型有：string | number | json | comma_delimited_list | boolean

### heat environment
Heat environment 中主要包含两个部分：

#### parameters：A list of key/value pairs.
#### resource_registry ：Definition of custom resources.

这个特性的好处是，当有更改时，只需要更改自定义文件的template 文件一个地方。

environment 有 user environment 和 global environment 两种，前者优先级高于后者。

heat.conf中定义了 environment_dir，默认是：/etc/heat/environment.d ，

当heat-engine起来的时候，会加载。可以将一些自定义的resource type放在此目录下作为global environment。

对于提供给客户的模版样例，个人推荐使用global enironment来增加自定义resource type，这样方便复用，对于用户来说的限制是，只能更改template中定义的parameter。

### Example use user environment
```
cat env.yaml

resource_registry:
  "OS::NestedServer": "file:///etc/heat/templates/nested.yaml"

cat server.yaml

heat_template_version: 2013-05-23
parameters:
  flavor:
    type: string
    default: m1.tiny
  image:
    type: string
    default: cirros-0.3.5-x86_64-disk
  network:
    type: string
    default: test-net
resources:
  server:
    type: OS::Heat::ResourceGroup
    properties:
      count: 1
      resource_def:
        type: nested.yaml
        properties:
          image: {get_param: image}
          flavor: {get_param: flavor}
          private_net: { get_param: network}
heat stack-create -f server.yaml -e env.yaml stack01
```
### Example use global environment
```
cat /etc/heat/environment.d/default.yaml

resource_registry:
...
...

"OS::NestedServer": "file:///etc/heat/templates/nested.yaml"

cat server-2.yaml

heat_template_version: 2013-05-23
parameters:
  flavor:
    type: string
    default: m1.tiny
  image:
    type: string
    default: cirros-0.3.5-x86_64-disk
  network:
    type: string
    default: test-net
resources:
  server:
    type: OS::Heat::ResourceGroup
    properties:
      count: 1
      resource_def:
        type: OS::NestedServer
        properties:
          image: {get_param: image}
          flavor: {get_param: flavor}
          private_net: { get_param: network}
```

`heat stack-create -f server-2.yaml stack02`

### template condition

### template composion

### composition

当需要写一个比较复杂的template的时候，往往需要将其分成几个小的template，然后再用template resource将这些合并成一个template。
用template 来定义resource，可以实现：

- 定义新的资源类型，构建自己的资源库
- 重新定义现有资源类型的行为

为了实现模版组合，heat 中会做如下工作：

- heatclient 拿到被引用的文件的内容，然后在调用heat-engine发 POST stacks/ api请求时，将其放在files section。
- heat-engine 中的environment 处理resource type 的mapping 和template 的创建
- heat-engine 将template 参数转换成resource属性。

template中的nested template 引用文件可以用如下几种方式：

- Relative path (my_nova.yaml)
- Absolute path (file:///home/user/templates/my_nova.yaml)
- Http URL (http://example.com/templates/my_nova.yaml)
- Https URL (https://example.com/templates/my_nova.yaml)

heat-engine 不会处理nested template中文件的内容，由heatclient来处理，原因是 heat-engine不会访问其所在的机器上的文件（due to security）,这样存在的一个问题是：

在horizon上创建stack，当template中含有get_file，或者包含子的template文件时，就会报错。

目前此问题已经被部分fix 了<mitaka>： https://review.openstack.org/#/c/241700/

fix 之后，horizon上的template可以使用nested template，但是只能使用http URL 或者 https URL.

将template file name 作为type
```
heat_template_version: 2015-04-30

resources:
  my_server:
    type: my_nova.yaml
    properties:
      key_name: my_key

```

定义新的resource type
```
resource_registry:
  "OS::Nova::Server": my_nova.yaml
```

这样做的好处是，当有多个template 文件用到这个resource type时，修改时只需要修改一个地方。

### netsted attributes

 要想得到 nested resource 的属性，可以通过 netsted templat 的 output 来得到。需要template version 在 2014-10-16 以上。

```
heat_template_version: 2015-04-30

resources:
my_server:
type: my_nova.yaml

outputs:
test_out:
value: {get_attr: my_server, resource.server, first_address}
```

### Making your template resource more “transparent”
Note Available since 2015.1 (Kilo).
If you wish to be able to return the ID of one of the inner resources instead of the nested stack’s identifier,
you can add the special reserved output OS::stack_id to your template resource
```
heat_template_version: 2015-04-30

resources:
  server:
    type: OS::Nova::Server

outputs:
  OS::stack_id:
    value: {get_resource: server}
```

Now when you use get_resource from the outer template heat will use the nova server id and not the template resource identifier.
heat resource type
OS::Heat::AutoScallingGroup
OS::Heat::ResourceGroup
using-heat-resourcegroup-resources

### index的使用
```
net_computes:
  depends_on: [api]
  type: OS::Heat::ResourceGroup
  properties:
    count: 2
    resource_def:
      type: ExternalResource
      properties:
        script: |
          #!/bin/bash
          portbrc=`echo $portip |awk -F '.' '{print $1"."$2"."$3".255"}'`
          type=("compute" "net")
          /opt/valor/scripts/expect.sh $user $member $vno $portip $portbrc
          # modify the user na  me in net.yml and compute.yml
          sed -i "s/ubuntu/$user/g" /opt/valor/scripts/net.sh
          sed -i "s/ubuntu/$user/g" /opt/valor/scripts/compute.sh
          for t in ${type[@]}
          do
              # run playbooks which defined in type
              /opt/valor/scripts/${t}.sh $portip
          done
        manager_id: {get_attr: [manager, server_id]}
        user: root
        vno: {get_attr: [net, vxlan]}
        network: {get_attr: [net, network_id]}
        index: '%index%'
        ips: {get_param: ips}
```
## Heat Notifications

openstack notification 用于通知用户或者开发者当前请求执行的状态

### heat enable notification

```
heat.conf: notification_driver=messagingv2
```
heat exchange and queues

```
|  source  |          destination   |    routing_key     |
+--------------+——————————————————————+———————————————---+
| heat     | notifications.error    | notifications.error|
| heat     | notifications.info     | notifications.info |

```
[template guide]:https://docs.openstack.org/heat/latest/

heat stack-list | grep stack- | awk '{system("heat stack-delete " $2)}'
