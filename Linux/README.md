# 一、网络编程

## 设置IP地址,网关DNS

```shell
vi  /etc/sysconfig/network-scripts/ifcfg-ens33 #Centos7.x编辑IP网关DNS相关信息涉及的文件

BOOTPROTO=static#启用静态IP地址
ONBOOT=yes  #开启自动启用网络连接
IPADDR=192.168.1.73   #设置网关
GATEWAY=192.168.1.1
NETMASK=255.255.255.0
DNS1=114.114.114.114
DNS2=8.8.8.8

service network restart

ip addr #Centos7.x版本查看ip
```



## 修改主机名

```shell
hostnamectl set-hostname  compute1
```

