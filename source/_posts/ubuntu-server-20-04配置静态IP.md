---
title: ubuntu-server-20.04配置静态IP
tags:
  - Ubuntu
date: 2024-11-08 21:22:53
---


- [一、查看网络信息](#一查看网络信息)
- [二、配置静态IP](#二配置静态ip)
- [三、参考文章](#三参考文章)

## 一、查看网络信息

`ip route`查看网关信息

```shell
whn@study:~$ ip route
default via 192.168.19.2 dev ens33 proto dhcp src 192.168.19.136 metric 100 
192.168.19.0/24 dev ens33 proto kernel scope link src 192.168.19.136 
192.168.19.2 dev ens33 proto dhcp scope link src 192.168.19.136 metric 100 
```

可以看到网关为`192.168.19.2`

虚拟机可以通过菜单栏中的`编辑`->`虚拟网络编辑器`->`NET设置`查看网关

{% asset_img "虚拟机查看网关1.png" "虚拟机查看网关1" %}

{% asset_img "虚拟机查看网关2.png" "虚拟机查看网关2" %}

{% asset_img "虚拟机查看网关3.png" "虚拟机查看网关3" %}

`ip a`查看网卡信息

```shell
whn@study:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:e9:07:33 brd ff:ff:ff:ff:ff:ff
    inet 192.168.19.136/24 brd 192.168.19.255 scope global dynamic ens33
       valid_lft 1694sec preferred_lft 1694sec
    inet6 fe80::20c:29ff:fee9:733/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到网卡为`ens33`，当前IP为`192.168.19.136/24`

## 二、配置静态IP

进入`/etc/netplan`，编辑配置文件`00-installer-config.yaml`

```shell
whn@study:~$ cd /etc/netplan/
whn@study:/etc/netplan$ sudo vim 00-installer-config.yaml
```

配置网卡`ens33`信息

默认配置

```yml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: true
  version: 2
```

修改后的配置

```yml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.19.100/24 # 静态IP和子网掩码
      routes:
        - to: default
          via: 192.168.19.2 # 网关
      nameservers: # DNS
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2
```

应用配置后的信息并查看网络信息

```shell
whn@study:~$ sudo netplan apply 
[sudo] password for whn: 
whn@study:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:e9:07:33 brd ff:ff:ff:ff:ff:ff
    inet 192.168.19.100/24 brd 192.168.19.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fee9:733/64 scope link 
       valid_lft forever preferred_lft forever
whn@study:~$ ip route
default via 192.168.19.2 dev ens33 proto static 
192.168.19.0/24 dev ens33 proto kernel scope link src 192.168.19.100
```

可以看到当前IP为`192.168.19.100`，修改成功

## 三、参考文章

ubuntu官网文章：

- Netplan文档：<https://netplan.readthedocs.io/en/stable/>

- Yaml配置文件说明：<https://netplan.readthedocs.io/en/stable/netplan-yaml/>

- 网络配置：<https://ubuntu.com/server/docs/configuring-networks>
