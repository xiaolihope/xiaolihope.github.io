2016/12/09

## OpenStack Restful API

```
REST 构建于http协议之上，Restful API 定义的标准方法：
GET：查询资源
POST：增加资源
PUT：更新资源
DELETE： 删除资源
HEAD: 验证，包括用户身份的验证和资源的验证。
```
在openstack中，所有的web服务都是通过wsgi部署的。

python web服务端程序开发：服务器程序（接收、整理客户端请求）＋应用程序（具体的逻辑处理）

### openstack 中的 Restful API

```
在wsgi中要实现url映射 -> 

Mapper : 实现url映射，当收到用户请求时，mapper根据url和方法来确定处理的方法。（在router包中定义好的）

Controller ：实现处理http请求的各种方法。（需要自己来实现）
```
## Paste + PasteDeploy + Routes + WebOb 来开发 wsgi 服务
```
PasteDeploy: 开发wsgi服务
    -> app (应用程序)
    -> filter
    -> pipeline
    -> composite

api-paste.ini: wsgi 服务的配置
```
## pecan + wsme

[pecan] : URL路由功能: 采用对象分发方式（object-dispatch style）进行URL 路由。

[WSME] : Web Service Made Easy  用来处理请求参数和响应内容的类型检查（英文简称就是typing）,专门解决web 服务的输入和输出类型检查。
通过装饰器来控制Controller 方法的输入和输出。

从config.py中： 'root': 'ironic.api.controllers.root.RootController', 看到入口是RootController。
只有被expose()装饰的方法才会暴漏给rest call。
pecan中有几个特殊的方法_lookup(), _default(), and _route()

RootController对应的是URL中根路径，也就是path中最左边的/。RootController继承自rest.RestController，
是Pecan实现的RESTful控制器。这里的get()函数表示，当访问的是GET /时，由该函数处理。
v1 = v1.Controller()表示，当访问的是GET /v1或者GET /v1/...时，请求由一个v1.Controller实例来处理

### URL Mapping

pecan 中的【url-mapping】

### 开发步骤
```
1.  初始化一个openstack 项目
2. Oslo-config-generator 生成config 文件: oslo-config-generator --config-file config-generator.conf
3. 初始化db：transformer-db-manage --config-dir /etc/transformer/transformer.conf upgrade head
    1. 创建user：MariaDB [(none)]> insert into mysql.user(Host,User,Password) values("localhost","transformer",password("transformer"));
                1. MariaDB [(none)]> flush privileges
    2. 创建db：create database transformer;  grant all privileges on transformer.* to transformer@localhost identified by 'transformer’; flush privileges;
    3. 创建table：
        ```markdown
        CREATE TABLE `tags` (
          `created_at` DATE ,
          `updated_at` DATE ,
          `project_id` char(80),
          `user_id` char(80) ,
          `id` int(11) NOT NULL,
        `is_default` boolean,
        `name` char(120) NOT NULL,
        `description` char(255),
          PRIMARY KEY (`id`)
        ) ;
        
        CREATE TABLE `tags` (`created_at` DATETIME ,   `updated_at` DATETIME ,   `project_id` char(80),   `user_id` char(80) ,   `id` int(2) NOT NULL  auto_increment , `is_default` boolean, `name` char(120) NOT NULL, `description` char(255),   PRIMARY KEY (`id`));
        
        alter table tags change id auto_increment;
        
        CREATE TABLE `associations` (
        `updated_at` DATE,
        `tag_id` int(11) NOT NULL,
        `resource_id` char(80),
        `resource_type` char(64),
        foreign key(tag_id) references tags(id) on delete cascade on update cascade
        );
        
        CREATE TABLE associations(`id` int(2) NOT NULL  auto_increment , updated_at DATETIME, tag_id int(11) NOT NULL, resource_id char(80), resource_type char(64),  foreign key(tag_id) references tags(id));
        
        CREATE TABLE associations(`id` int(2)  NOT NULL auto_increment, updated_at DATETIME,  `created_at` DATETIME , tag_id int(11) NOT NULL, resource_id char(36), resource_type char(64),  foreign key(tag_id) references tags(id),  PRIMARY KEY (`id`));
        ```

4. 启动api service：`transformer-api --config-file /etc/transformer/transformer.conf`
5. 测试api
    ```markdown
    1.  curl -g -i -X GET http://localhost:8787/v1/tags -H "Accept: application/json" 
     ```
6. 和keystone整合
```
## References

openstack api服务blog from unitedstack

[api-1] , [api-2], [api-3] , [api-4]

[api-1]: https://www.ustack.com/blog/demoapi1/
[api-2]: https://www.ustack.com/blog/demoapi2/
[api-3]: https://segmentfault.com/a/1190000003810294
[api-4]: https://segmentfault.com/a/1190000004004179


[pecan]: http://pecan.readthedocs.io/en/latest/
[WSME]: https://pythonhosted.org/WSME/
[url-mapping]: http://pecan.readthedocs.io/en/latest/rest.html#url-mapping

