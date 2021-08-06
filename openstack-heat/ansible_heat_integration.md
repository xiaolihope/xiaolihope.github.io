# Ansible Heat Integration

2017/04/05

本文来谈谈Heat和Ansible合作，实现更灵活的部署和编排。 我们知道，heat是openstack中用于编排的模块，它通过将OpenStack中的资源 （resource）以模版（template）的形式组织起来，我们可以将一组资源，比如虚拟机实例的启动、IP绑定、软件部署等写在一个template里面，heat通过读取配置文件来完成模版规定的动作：创建虚拟机，associate floatingip，deploy application 等等。  
Heat将从这个template中创建出来的一组资源称之为“资源栈”(stack)。当对这一组资源进行操作时，只需要对stack进行操作，所以heat很适合批量资源的创建和销毁。它将一系列繁琐的人工操作自动化了起来。 
Heat Orchestration 包含2层含义：provisioning and deployment.

除了资源的部署之外，还有一方面是server上应用软件的安装配置。总结一下，在heat中大体有如下三种方式可以控制server上的配置：

1. 定制image
2. user-data boot scripts and cloud-init
3. software deployment resources

对于heat中提供的sofware config和software deployment 使用起来的缺点是：部署遇到问题，debug比较麻烦。

Ansible 是一个很好的部署工具，因此我们可以将ansible和heat结合起来，实现更加灵活、方便开发和debug的部署编排。

## 1. Ansible Tower 和 Heat 集成

## 2. Ansible Core 和 Heat 集成

### Deployment Model

一个应用由一个(一些)autoscalling groups组来构成，每个autoscalling group组由由一个或多个节点组成。为了更好的部署和维护应用，
我们引入了一个 manager node, 如下图：

```
                       /-- node 1
           /-- asg 1 --|-- node 2 ..
           |           |-- ....
manager -- |-- asg 2   \-- node N ..
           |-- asg 3
           |-- ....
           |-- asg N
               (scale up, scale down)

```

### 自定义resource types

`heat-resources-customized` project提供了一个共享的自定义heat resource type，这些自定义的资源类型用于更方便地部署/维护复杂的应用编排。

新增resource type 如下：
```
resource_registry:
  "SelfDefined::Heat::Config": "../templates/polex_heat_config.yaml"
  "SelfDefined::Heat::Deploy": "../templates/polex_heat_deploy.yaml"
  "SelfDefined::Heat::Keypair": "../templates/polex_heat_keypair.yaml"
  "SelfDefined::Heat::Nova::Server": "../templates/polex_heat_nova_server.yaml"
  "SelfDefined::Heat::Nova::Server2": "../templates/polex_heat_nova_server2.yaml"
  "SelfDefined::Heat::Port": "../templates/polex_heat_port.yaml"
  "SelfDefined::Heat::Net": "../templates/polex_heat_net.yaml"
  "SelfDefined::Heat::FloatingIP": "../templates/polex_heat_floatingip.yaml"
  "SelfDefined::Heat::Snippet": "../templates/polex_heat_snippet.yaml"
  "SelfDefined::Heat::Snippet2": "../templates/polex_heat_snippet2.yaml"
  "SelfDefined::Heat::Member": "../templates/polex_heat_member.yaml"
  "SelfDefined::Heat::Member2": "../templates/polex_heat_member2.yaml"
  "SelfDefined::Heat::Manager": "../templates/polex_heat_manager.yaml"
  "SelfDefined::Heat::Member::Config": "../templates/polex_heat_member_config.yaml"
  "SelfDefined::Heat::Group": "../templates/polex_heat_group.yaml"
  "SelfDefined::Heat::Group2": "../templates/polex_heat_group2.yaml"
  "SelfDefined::Heat::ExternalResource": "../templates/polex_heat_external_resource.yaml"
```
#### `SelfDefined::Heat::Manager`
```
heat_template_version: 2015-04-30

parameters:
  network:
    description: network
    type: string
  config_id:
    description: config id
    type: string
  secgroup_id:
    description: security group id
    type: string
  image:
    description: image
    type: string
  flavor:
    description: flavor
    type: string
    default: m1.small
  keypair:
    description: keypair id
    type: string
  user_data:
    description: user data
    type: string
    default: ""
  snippet:
    description: snippet
    type: json
    default: {}

resources:
  p:
    type: SelfDefined::Heat::Port
    properties:
      security_groups:
        - get_param: secgroup_id
      allowed_address_pairs:
        - ip_address: "0.0.0.0/0"
      network: {get_param: network}

  m:
    type: SelfDefined::Heat::Nova::Server
    properties:
      image: {get_param: image}
      flavor: {get_param: flavor}
      keypair: {get_param: keypair}
      networks:
        - port: {get_attr: [p, port_id]}
      user_data: {get_param: user_data}
      settings:
        - {get_param: [snippet, set_static_ip, enabled]}
        - {get_param: [snippet, os_collect_config, enabled]}
  d:
    type: SelfDefined::Het::Deploy
    properties:
      config_id: {get_param: config_id}
      server_id: {get_attr: [m, server_id]}
      inputs:
        member_id: {get_attr: [m, server_id]}
        member_ip: {get_attr: [m, server_ip]}
        member_restart_url: {get_attr: [m, restart_url]}

outputs:
  server_id:
    value: {get_attr: [m, server_id]}
    description: server id
  server_ip:
    value: {get_attr: [m, server_ip]}
  server_port_id:
    value: {get_attr: [p, port_id]}
    description: server port id
  restart_url:
    description: url to restart the instance.
    value: {get_attr: [m, restart_url]}
```
#### `SelfDefined::Heat::ExternalResource`
```
heat_template_version: 2013-05-23
parameters:
  script:
    description: shell script
    type: string
  manager_id:
    description: manager id
    type: string
  user:
    description: user name
    type: string
  network:
    description: network
    type: string
  vno:
    description: vxlan segmentation id
    type: string
  ips:
    description: member ip
    type: comma_delimited_list
  index:
    type: number

resources:
  external_port:
    type: SelfDefined::Heat::Port
    properties:
      network: {get_param: network}
  config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: script
      inputs:
      - name: member
      - name: user
      - name: vno
      - name: portip
      config: {get_param: script}
  deployment:
    type: OS::Heat::SoftwareDeployment
    properties:
      config:
        get_resource: config
      server: {get_param: manager_id}
      input_values:
        member: {get_param: [ips, {get_param: index}]}
        user: {get_param: user}
        vno: {get_param: vno}
        portip: {get_attr: [external_port, port_ip]}

outputs:
  stdout:
    value:
      get_attr: [deployment, deploy_stdout]
  stderr:
    value:
      get_attr: [deployment, deploy_stderr]
  status_code:
    value:
      get_attr: [deployment, deploy_status_code]
  port_ip:
    value:
      get_attr: [external_port, port_ip]
  port_id:
    value:
      get_attr: [external_port, port_id]
  port_mac:
    value:
      get_attr: [external_port, port_mac]
  config_id:
    value: {get_resource: config}
```

#### `SelfDefined::Heat::Group`
```
heat_template_version: 2015-04-30
parameters:
  name:
    description: group name
    type: string
  desired_capacity:
    description: desired capacity
    type: number
    default: 1
  network:
    description: network
    type: string
  image:
    description: image
    type: string
  flavor:
    description: flavor
    type: string
    default: m1.small
  keypair:
    description: keypair id
    type: string
  user_data:
    description: user data
    type: string
    default: ""
  snippet:
    description: snippet
    type: json
    default: {}

resources:
  g:
    type: OS::Heat::AutoScalingGroup
    properties:
      resource:
        type: SelfDefined::Heat::Member2
        properties:
          keypair: {get_param: keypair}
          network: {get_param: network}
          image: {get_param: image}
          flavor: {get_param: flavor}
          user_data: {get_param: user_data}
          snippet: {get_param: snippet}
          group: {get_param: name}
      min_size: 1
      desired_capacity: {get_param: desired_capacity}
      max_size: 250
  u:
    type: OS::Heat::ScalingPolicy
    properties:
      adjustment_type: change_in_capacity
      auto_scaling_group_id: {get_resource: g}
      cooldown: 60
      scaling_adjustment: 1
  d:
    type: OS::Heat::ScalingPolicy
    properties:
      adjustment_type: change_in_capacity
      auto_scaling_group_id: {get_resource: g}
      cooldown: 60
      scaling_adjustment: -1
outputs:
  scale_up_url:
    value: {get_attr: [u, alarm_url]}
  scale_down_url:
    value: {get_attr: [d, alarm_url]}
  current_size:
    value: {get_attr: [g, current_size]}
  ips:
    value: {get_attr: [g, outputs_list, server_ip]}
```

### 部署模版
`heat-orchestrate-application` project提供了部署application的ansible playbok roles和heat部署模版以及部署需要的脚本。
```
heat_template_version: 2015-04-30
#
# Heat OpenStack Template - HOST
#
parameters:
  external_network:
    type: string
    default: public
  internal_network:
    type: string
    default: priv-net
  manager_image:
    type: string
    default: centos-7.2
  image:
    type: string
    default: centos-7.2
  flavor:
    type: string
    default: m1.medium
resources:
  secgroup:
    type: secgroup.yaml
  keypair:
    type: SelfDefined::Heat::Keypair
  snippet:
    type: SelfDefined::Heat::Snippet
  config:
    type: SelfDefined::Heat::Member::Config
    properties:
      script:
        str_replace:
          template: |
            #!/bin/bash
            mkdir -p /opt/heater
            mkdir -p /root/.ssh
            cat > /root/.ssh/id_rsa << EOF
            id_rsa_contents
            EOF
            cd /opt/heater
            (cat | base64 -d | gunzip | cpio -id) << EOF
            deploy_contents
            EOF
            chown -R root:root /opt/heater
            apt-get install -y software-properties-common
            apt-add-repository ppa:ansible/ansible
            apt-get update
            apt-get install -y ansible

            chmod 600 /root/.ssh/id_rsa
          params:
            id_rsa_contents: {get_attr: [keypair, private_key]}
            deploy_contents: {get_attr: [deploy_contents, contents]}
  deploy_contents:
    type: deploy_contents.yaml
  manager:
    type: SelfDefined::Heat::Manager
    properties:
      network: {get_param: internal_network}
      config_id: {get_attr: [config, config_id]}
      secgroup_id: {get_attr: [secgroup, mgm_sg_id]}
      image: {get_param: manager_image}
      keypair: {get_attr: [keypair, keypair_id]}
      snippet: {get_attr: [snippet, snippet]}
  db:
    depends_on: [manager]
    type: SelfDefined::Heat::Group
    properties:
      manager_id: {get_attr: [manager, server_id]}
      network: {get_param: internal_network}
      secgroup_id: {get_attr: [secgroup, db_sg_id]}
      image: {get_param: image}
      flavor: {get_param: flavor}
      keypair: {get_attr: [keypair, keypair_id]}
      snippet: {get_attr: [snippet, snippet]}
      script: |
        #!/bin/bash
        /opt/heater/scripts/db.sh ${member_ip}
      desired_capacity: 1
```
cat `deploy_contents.yaml`
```
#!/bin/bash
cd `dirname ${BASH_SOURCE[0]}`/..
(
    echo "heat_template_version: 2013-05-23"
    echo "outputs:"
    echo "  contents:"
    echo "    description: deploy contents"
    echo "    value: |"
    find * | sort | cpio -oc | gzip -9 -n | base64 | while read line; do
        echo "      $line";
    done
) > heat/deploy_contents.yaml
```
cat `db.sh`
```
function append_host {
    local ip=$1
    local index=$2
    local group=$3
    local user=$4
    local host=$5
    [ -z "$host" ] && host=${environment}-${group}${index}
    cat > $inventory/$host << EOF
[$group]
$host ansible_user=$user ansible_host=$ip ansible_ssh_private_key_file=$id_rsa
EOF
cat > $host_vars/$host << EOF
hostname: $host
management_ip_address: $ip
EOF
ansible-playbook -i $inventory db.yaml
```

## Reference
- [Rackspace Orchestration document](https://developer.rackspace.com/docs/user-guides/orchestration/ansible/using-ansible-with-heat/)
- [ansible with heat](https://keithtenzer.com/2016/05/09/openstack-heat-and-ansible-automation-born-in-the-cloud/)
- [ansible heat code](https://github.com/ktenzer/openstack-heat-templates/blob/master/ansible/centos-tower.yaml)
- [Full stack Automation with ansible and openstack](https://github.com/ktenzer/openstack-heat-templates/blob/master/ansible/centos-tower.yaml)
