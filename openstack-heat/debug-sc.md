*date: 2017-04-06T18:16:32+08:00*

The general practices to debug a user-provided script at SoftwareConfig: 

1. For cloud-init debug 
nova console-log 

2. For os-* tools debug: 
step1: 登录到server上查看script是否以及被执行。 （根据执行结果来判断） 
stwp2: 如果没有执行，那需要查看是哪个环节出了问题。 

a. we can debug the os-collect-config using command:
os-collect-config --force --one-time --debug

b. You can also go to the /var/lib/heat-config/heat-config-script and execute your script (it is visible in this directory as file (with ID as name) e.g. 6444.....
c. answer from steve baker

Currently if a deployment resource remains IN_PROGRESS (then times out) the only way to diagnose the issue is to ssh into your server and trigger os-collect-config manually to see what is actually happening. To do this, ssh into your server and run the following:
```bash
  # sudo service os-collect-config stop
  # sudo os-collect-config --force --one-time --debug
```
This will do a full metadata poll and then eventually run /usr/libexec/os-refresh-config/configure.d/55-heat-config which will in turn run the specified hook with your deployment data. 
Generally you should see if there is a problem with your config, or with the networking to signal back to heat. 
Throughout the Kilo development cycle there will be many improvements to the debuggability of software deployment resources. 
Some of these improvements will happen in the hooks or Python-heatclient, so they will also be available in Juno and Icehouse heat. I’ll update this answer as they become available. 
—part log 

Running /usr/libexec/os-refresh-config/configure.d/55-heat-config 
```bash
[2016-07-07 10:20:52,313] (heat-config) [WARNING] Skipping config 18ca89ca-8b91-4830-b3af-15dbe6f85fb4, already deployed 
[2016-07-07 10:20:52,314] (heat-config) [WARNING] To force-deploy, rm /var/run/heat-config/deployed/18ca89ca-8b91-4830-b3af-15dbe6f85fb4.json 
[2016-07-07 10:20:52,314] (heat-config) [WARNING] Skipping config 90210905-531f-4312-84a6-50fd54e1b203, already deployed 
[2016-07-07 10:20:52,314] (heat-config) [WARNING] To force-deploy, rm /var/run/heat-config/deployed/90210905-531f-4312-84a6-50fd54e1b203.json 
dib-run-parts Thu Jul 7 10:20:52 UTC 2016 55-heat-config completed
```

Ref link: https://fatmin.com/2016/02/23/openstack-heat-and-os-collect-config/
1. os-collect-config
link: https://wiki.openstack.org/wiki/OsCollectConfig
结构图

![os-tools](../images/os-tools.png)
os-refresh-config

os-apply-config

os-notify-config