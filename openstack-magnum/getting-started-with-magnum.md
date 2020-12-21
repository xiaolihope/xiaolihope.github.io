## 一、概念
1. cluster，k8s or swarm 的集群，以前叫bay
     1.1 cluster中的infrastructure包含：
     
          a. servers: vm/bm
          b. Identity:
          c. Network: kuryr 
          d. storage: cinder
          E: security barbican
     1.2 cluster的life cycle:CRUD
          
2. clusterTemplate: 定义docker集群中的一些规格，比如集群管理节点的flavor or 计算节点的flavor or 集群使用的image等等；或者coe的一些属性；当有cluster使用时，不可以删除或者更新；由下面三部分组成
     
     a.  mandatory
     
          Name:可以重复
          -coe 必填 kubernetes/swarm/mesos
          -image: 必须含有os_distro 属性，区分大小写。kubernetes:fedora-atomic, coreos
          -keypair 用来登陆到集群中的节点
          -external-network：用于集群连接外网，下载image等。
          —public 默认是非公有，只对admin或者owner可见
          -server-type：default is vm, can be vm/bm
          -network-driver for kubernetes the default network is flannel. Can be flannel/calico // 区分大小写
          -volume-driver:for kubernetes is cinder//区分大小写
          -docker-storage-deiver images和容器写层面的driver名字，默认为devicemapper
          -docker-volume-size对于devicemapper，最小是3g。如果指定则数据存在cinder的卷中，否则存在计算节点的本地盘中；
          -lables <key:value> 集群driver 额外传递参数信息
          —tls-disabled. 默认是enable，在部署遇到问题的时候，disable可以不用证书就登陆coe endpoints
           —registry-enabled: 默认是从公有仓库下载image，也可以指定私有的。
          —master-lb-enabled: 默认是true。当有多个master时，创建load balancer 提供api并且将请求发给masters。当load balancer不可用时，可以禁用，其中一个master的ip将作为api endpoint。
            -discovery-url 各个coe不同
            -timeout 默认是60分钟 
                             
     b. infrastructure
     
     c. Coe specific
     
3. node：集群中的某个节点
4. Container: 具体某个docker容器
5. COE: container orchestration engine, 容器集群管理后端，magnum的工作就是将用户的请求转发到coe，完成容器的管理。
6. pod：一个pod会包含n+1个container，多出来的是net container，专门做路由的。
7. service： 理解为pod的一个路由（网关），因为pod在运行中可能被删除或者ip发生变化，service可以保证pod的动态变化对访问端是透明的；
8. replication controller：是pod的复制抽象。通过它用户可以指定一个应用需要几份复制。

## 二、主要服务

1. magnum-api：将client端的请求通过消息队列发送到后端。

2. Magnum-conductor: magnum支持的coe有k8s, swarm, docker, conductor负责将请求转发到对应的coe

## 三、magnun 使用流程

1. 创建clustertemplate 定义集群的规格
2. 创建k8s/mesos/swarm cluster
3. 通过magnum api 或者后台的k8s命令创建container

## 四、cli 列表
1. Cluster-template-create/delete/list/show/update
2. Cluster-config/create/delete/list/show/update
     2.1 openstack coe cluster create my cluster —cluster-template my template —node-count 8 —master-count 3
3. quotas/create/delete/list/show/update
4. Ca-rotate/show/sign
5. service-list
6. Stats-list: // 展示cluster和nodes 的个数

## 五、使用注意事项
1.  image should contains Docker and COE(kubnetes/swarm/mesos)；特制的image和magnun中以下相配合：
          heat-templates
          clusterTemplate和heat template parameters的模版定义映射
          配置软件的脚本
          image需要支持cloud-init和heat software config的配置工具：os-collect-config/ os-refresh-config/ os-apply-config /heat-config /heat-config-script
     
2. Magnum 没有提供额外的api，部署完cluster之后，需要用native client（cli or web）和cluster交互；注意clietversion要和clusterversion匹配；
     coe 层面：kubernetes，swarm，mesos等命令
     容器层面：docker 适用于所有的clusters
3. neutron需要配置lbaas v1 or v2, 并且k8s 创建的neutron 相关的资源不被heat管理，所以删除集群时，需要先手动删除neutron lb相关的资源，再删除stack。否则会导致heat stack 删除失败。
4. 使用流程
          4.1 创建cluster-template
          4.2 创建cluster
          4.3 更新node 个数 //建议在减少node个数时使用replication controller，以使被删除的pod可以在现存的节点上恢复。
## 六、 kubernetes cli
1. cli：
          
curl -O https://storage.googleapis.com/kubernetes-release/release/v1.2.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
2. web：
eval $(openstack coe cluster config <cluster-name>)
kubectl proxy
The browser can be accessed at http://localhost:8001/ui
## 七、cluster networking
Flannel: host-gw 
## 八、magnum手动安装笔记
1. 创建db
```markdown
# mysql -uroot -ptuscloud
CREATE DATABASE magnum;
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'localhost' \
-> IDENTIFIED BY 'magnum';
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%' \
-> IDENTIFIED BY 'magnum';
Query OK, 0 rows affected (0.00 sec)
```

2. keystone中创建相关资源
```markdown
160 . openrcv3
161 openstack user create --domain default --password-prompt magnum
162 openstack role add --project service --user magnum admin
163 openstack service create --name magnum --description "OpenStack Container Infrastructure Management Service" container-infra
164 openstack endpoint create --region RegionOne container-infra public http://172.17.0.3:9511/v1
165 openstack endpoint create --region RegionOne container-infra internal http://172.17.0.3:9511/v1
166 openstack endpoint create --region RegionOne container-infra admin http://172.17.0.3:9511/v1
167 openstack domain create --description "Owns users and projects \
created by magnum" magnum
168 openstack user create --domain magnum --password-prompt magnum_domain_admin
169 openstack role add --domain magnum --user-domain magnum --user magnum_domain_admin admin
```

3. 安装 magnum 
进入到docker中
```markdown
# yum install centos-release-openstack-rocky -y
# yum install openstack-magnum-api openstack-magnum-conductor python-magnumclient

```
4. 配置magnum 文件
```markdown
-rw-r----- 1 root magnum 74315 May 15 15:25 magnum.conf
```

5. Populate db: 
```markdown
su -s /bin/sh -c "magnum-db-manage upgrade" magnum
```

6. 用screen 启动magnum 服务
```markdown
/usr/bin/python /usr/bin/magnum-api —cogfile=/var/log/magnum/api.log
/usr/bin/python /usr/bin/magnum-conductor —logfile=/var/log/magnum/conductor.log
```

7. Quick start
https://docs.openstack.org/magnum/rocky/contributor/quickstart.html
8. Using magnum
```markdown
* Step1: upload image from tuscloud manager ui, and then update image by:
# bzip2 -dk coreos_production_openstack_image.img.bz2
# qemu-img convert -f raw -O qcow2 coreos_production_openstack_image.img coreos_production_openstack_image.qcow2
# glance image-update --os-distro coreos 2ff2404e-6ed5-4c65-81ed-09d7928d5f97. //fedora-atomic 如果是fedora则设置成这个值
If not, will met the following issue:# openstack coe cluster template create --coe kubernetes --image magnum-coreos --external-network public k8s-cluster-template-coreos --docker-volume-size 3 --server-type vm
Cluster type (vm, None, kubernetes) not supported (HTTP 400) (Request-ID: req-e6c63697-55ad-4658-b7d3-d22f364b0990)
* Step2: create cluster template:
$ openstack coe cluster template create --coe kubernetes --image magnum-coreos --external-network public k8s-cluster-template-coreos --docker-volume-size 3 --server-type vm --flavor 0134be66-1d98-4c8b-aad2-642aa8b56fbc --keypair dxl-key
* Step3: create cluster
$ openstack coe cluster create --cluster-template k8s-cluster-template-coreos --node-count 1 k8s-cluster
我们的环境中server-group不支持policy
m1.small
Error1: ERROR: Required microversion for soft policies not supported.
Fix: 从magnum端，去掉创建server group时的policy部分
/usr/lib/python2.7/site-packages/magnum/drivers
./magnum.conf:#nodes_affinity_policy = soft-anti-affinity —> anti-affinity
Error2: Cluster error, stack status: CREATE_FAILED, stack_id: 84eb57bc-dc12-4b43-ae7b-4127617787ab, reason: Resource CREATE failed: resources[0]: resources.kube_masters.Property error: resources.docker_volume.properties.volume_type: Error validating value '': The VolumeType () could not be found.
Fix:
./magnum.conf:#default_docker_volume_type —>
./magnum.conf:default_docker_volume_type =<volume-type> // first create one local disk type on manager. Not worked skip it first. Comment out the cinder volume and attach resource.
cerror3: 
```
## 九、magnum代码分析
9.1 cluster 创建过程分析
1. api接受创建cluster的api请求
```markdown
dixiaoli-repos/magnum/magnum/api/controllers/v1/bay.py
@base.Controller.api_version("1.1", "1.1")
@expose.expose(Bay, body=Bay, status_code=201)
def post(self, bay):
    """Create a new bay.
    :param bay: a bay within the request body.
    """
    new_bay = self._post(bay)
    res_bay = pecan.request.rpcapi.cluster_create(new_bay,
                                                  bay.bay_create_timeout)
    # Set the HTTP Location Header
    pecan.response.location = link.build_url('bays', res_bay.uuid)
    return Bay.convert_with_links(res_bay)
这里会判断创建的参数，docker_volume_size, labels, master_flavor_id, flavor_id 如果没有，则用cluster template中的值。
```

2. 请求通过rpc转发给conductor
dixiaoli-repos/magnum/magnum/conductor/handlers/cluster_conductor.py
```markdown
def cluster_create(self, context, cluster, create_timeout):
    LOG.debug('cluster_heat cluster_create')
    osc = clients.OpenStackClients(context)
    cluster.status = fields.ClusterStatus.CREATE_IN_PROGRESS
    cluster.status_reason = None
    cluster.create()
    try:
        # Create trustee/trust and set them to cluster
        trust_manager.create_trustee_and_trust(osc, cluster)
        # Generate certificate and set the cert reference to cluster
        cert_manager.generate_certificates_to_cluster(cluster,
                                                      context=context)
        conductor_utils.notify_about_cluster_operation(
            context, taxonomy.ACTION_CREATE, taxonomy.OUTCOME_PENDING)
        # Get driver
        cluster_driver = driver.Driver.get_driver_for_cluster(context,
                                                              cluster)
        # Create cluster
        cluster_driver.create_cluster(context, cluster, create_timeout)
        cluster.save()
    except Exception as e:
        cluster.status = fields.ClusterStatus.CREATE_FAILED
        cluster.status_reason = six.text_type(e)
        cluster.save()
        conductor_utils.notify_about_cluster_operation(
            context, taxonomy.ACTION_CREATE, taxonomy.OUTCOME_FAILURE)
        if isinstance(e, exc.HTTPBadRequest):
            e = exception.InvalidParameterValue(message=six.text_type(e))
            raise e
        raise

    return cluster
```

Conductor 会被bay 创建密钥，然后拿到cluster heat driver，然后调用 heat driver 的create_cluster去创建
3. 调用heat，创建stack
dixiaoli-repos/magnum/magnum/drivers/heat/driver.py
```markdown
def create_cluster(self, context, cluster, cluster_create_timeout):
    stack = self._create_stack(context, clients.OpenStackClients(context),
                               cluster, cluster_create_timeout)
    # TODO(randall): keeping this for now to reduce/eliminate data
    # migration. Should probably come up with something more generic in
    # the future once actual non-heat-based drivers are implemented.
    cluster.stack_id = stack['stack']['id']

def _create_stack(self, context, osc, cluster, cluster_create_timeout):
    template_path, heat_params, env_files = (
        self._extract_template_definition(context, cluster))
    tpl_files, template = template_utils.get_template_contents(
        template_path)
    environment_files, env_map = self._get_env_files(template_path,
                                                     env_files)
    tpl_files.update(env_map)
    # Make sure we end up with a valid hostname
    valid_chars = set(ascii_letters + digits + '-')
    # valid hostnames are 63 chars long, leaving enough room
    # to add the random id (for uniqueness)
    stack_name = cluster.name[:30]
    stack_name = stack_name.replace('_', '-')
    stack_name = stack_name.replace('.', '-')
    stack_name = ''.join(filter(valid_chars.__contains__, stack_name))
    # Make sure no duplicate stack name
    stack_name = '%s-%s' % (stack_name, short_id.generate_id())
    stack_name = stack_name.lower()
    if cluster_create_timeout:
        heat_timeout = cluster_create_timeout
    else:
        # no cluster_create_timeout value was passed in to the request
        # so falling back on configuration file value
        heat_timeout = cfg.CONF.cluster_heat.create_timeout
    fields = {
        'stack_name': stack_name,
        'parameters': heat_params,
        'environment_files': environment_files,
        'template': template,
        'files': tpl_files,
        'timeout_mins': heat_timeout
    }
    created_stack = osc.heat().stacks.create(**fields)
    return created_stack

def _extract_template_definition(self, context, cluster,
                                 scale_manager=None):
    cluster_template = conductor_utils.retrieve_cluster_template(context,
                                                                 cluster)
    definition = self.get_template_definition()
    return definition.extract_definition(context, cluster_template,
                                         cluster,
                                         scale_manager=scale_manager)

from magnum.drivers.k8s_fedora_atomic_v1 import template_def

def get_template_definition(self):
    return template_def.AtomicK8sTemplateDefinition()

class AtomicK8sTemplateDefinition(kftd.K8sFedoraTemplateDefinition):
    """Kubernetes template for a Fedora Atomic VM."""
    @property
    def driver_module_path(self):
        return __name__[:__name__.rindex('.')]
    @property
    def template_path(self):
        return os.path.join(os.path.dirname(os.path.realpath(__file__)),
                            'templates/kubecluster.yaml')
```

4. 调用heat 创建stack之前需要先构造参数
dixiaoli-repos/magnum/magnum/drivers/heat/k8s_coreos_template_def.py
```markdown
def get_env_files(self, cluster_template, cluster):
    env_files = []
    # template_def.add_priv_net_env_file(env_files, cluster_template)
    # template_def.add_etcd_volume_env_file(env_files, cluster_template)
    # template_def.add_volume_env_file(env_files, cluster)
    # template_def.add_lb_env_file(env_files, cluster_template)
    # template_def.add_fip_env_file(env_files, cluster_template)

    return env_files
dixiaoli-repos/magnum/magnum/drivers/heat/template_def.py

def add_priv_net_env_file(env_files, cluster_template):
    if cluster_template.fixed_network:
        env_files.append(COMMON_ENV_PATH + 'no_private_network.yaml')
    else:
        env_files.append(COMMON_ENV_PATH + 'with_private_network.yaml')
```


十、创建cluster，新增nova server的boot option参数，从创建cluster，labels传入，如下：
```markdown
openstack coe cluster create --cluster-template k8s-cluster-template-coreos --docker-volume-size 3 --keypair dxl-key --master-count 1 --node-count 1 --flavor 0134be66-1d98-4c8b-aad2-642aa8b56fbc --labels instance_store_type=local,instance_store_uuid=a3239e52-2902-44ac-81ff-3c28ead35e98,image_store_type=local,image_store_uuid=a3239e52-2902-44ac-81ff-3c28ead35e98 k8s-c1
```