---
date: 2017-02-27T17:51:32+08:00
title: linux operation
---

## vnc
```
On server sice:
    run #vncserver
on the client sice:
    run #vncviewer
    then type: 9.119.148.198:1
Done
```
## upgrade 14.04 to 16.04
[upgrade 1404 to 1604] can do the following:

```
    # sudo update-manager -d
    # sudo apt-get update && sudo apt-get dist-upgrade
    # sudo init 6
    # sudo apt-get install update-manager-core
    # sudo vi /etc/update-manager/release-upgrades
    # sudo do-release-upgrade -d
```
## linux login without private key
```
    # ssh-keygen
    # ssh-copy-id -i id_rsa.pub root@9.119.148.198
```

[upgrade 1404 to 1604]: http://www.tecmint.com/upgrade-ubuntu-14-04-to-16-04/


## Linux 创建 virtualenv

### virtualenv 的作用

virtualenv用于创建独立的Python环境，多个Python相互独立，互不影响，它能够：

1. 在没有权限的情况下安装新套件

2. 不同应用可以使用不同的套件版本

3. 套件升级不影响其他应用

### 如何创建&使用

```
# yum install -y python-virtualenv
# virtualenv env_transformer
# . env_transformer/bin/activate
# deactivate

```

默认情况下，虚拟环境会依赖系统环境中的site package,就是说系统中已经安装好的第三方package也会安装在虚拟环境中，如果不想依赖这些package，
那么可以加上参数 `no-site-packages` eg: `virtualenv --no-site-packages [env_name]`


在虚拟环境中安装python套件时，如果没有没有启动虚拟环境，可以在 `~/.bashrc` 文件中加上：`export PIP_REQUIRE_VIRTUALENV=true` or
`export PIP_REQUIRE_VIRTUALENV=true` 这样就可以安装到虚拟环境中。

## Virtualenvwrapper

Virtaulenvwrapper是virtualenv的扩展包，用于更方便管理虚拟环境，它可以做：

1. 将所有虚拟环境整合在一个目录下

2. 管理（新增，删除，复制）虚拟环境

3. 切换虚拟环境

4. ...

### 安装 & 使用

```
# pip install virtualenvwrapper

```

`which virtualenvwrapper.sh` 找到安装目录为 `/usr/bin/virtualenvwrapper.sh` 打开这个文件可以看到安装步骤如下：

### Setup:

```
#  1. Create a directory to hold the virtual environments.
#     (mkdir $HOME/.virtualenvs).
#  2. Add a line like "export WORKON_HOME=$HOME/.virtualenvs"
#     to your .bashrc.
#  3. Add a line like "source /path/to/this/file/virtualenvwrapper.sh"
#     to your .bashrc.
#  4. Run: source ~/.bashrc
#  5. Run: workon
#  6. A list of environments, empty, is printed.
#  7. Run: mkvirtualenv temp
#  8. Run: workon
#  9. This time, the "temp" environment is included.
# 10. Run: workon temp
# 11. The virtual environment is activated.
```
#### 列出env

`workon`

`lsvirtualenv`

#### 启动／切换虚拟环境

`workon [env_name]`

#### 删除虚拟环境

`rmvirtualenv [env_name]`

#### 离开虚拟环境

`deactivate`


## iptables

```commandline
$ iptables-save

$ iptables -t nat -A POSTROUTING -s 172.24.4.0/24 ! -d 172.24.4.0/24 -j MASQUERADE
```

## Linux 增加磁盘

```
# lsblk
# fdisk /dev/sdb
n
w
# mkfs.ext4 /dev/sdb
```

## Linux server安装GUI

```
# yum groupinstall "GNOME Desktop" "Graphical Administration Tools"
# ln -sf /lib/systemd/system/runlevel5.target /etc/systemd/system/default.target
# reboot
```