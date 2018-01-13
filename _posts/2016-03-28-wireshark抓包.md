---
layout:     post
title:      "Wireshark 抓包"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- 网络
- 抓包
---
# 一、安装

```java
sudo add-apt-repository ppa:wireshark-dev/stable
sudo apt-get install wireshark
```
打开 wiresshark 出现错误： `Can't run /usr/bin/dumpcap in child process Permission denied`

解决方法：

```java
sudo chmod +x /usr/bin/dumpcap
```
# 二、使用

## 2.1 ifconfig 命令
ifconfig 命令用来查看和配置网络设备。当网络环境发生改变时可通过此命令对网络进行相应的配置。
```java
ifconfig

enp1s0    Link encap:以太网  硬件地址 28:x2:44:84:57:xx  
          inet 地址:222.20.30.15  广播:222.20.31.255  掩码:255.255.254.0
          inet6 地址: fe80::28b0:5eab:d46c:7b87/64 Scope:Link
          inet6 地址: 2001:250:4000:821e:d0c9:18ee:f91f:2e1/64 Scope:Global
          inet6 地址: 2001:250:4000:821e:c5a5:37ad:eff7:56b5/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:12882 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:8896 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000
          接收字节:5986239 (5.9 MB)  发送字节:1653703 (1.6 MB)

lo        Link encap:本地环回  
          inet 地址:127.0.0.1  掩码:255.0.0.0
          inet6 地址: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  跃点数:1
          接收数据包:4824 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:4824 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1
          接收字节:588324 (588.3 KB)  发送字节:588324 (588.3 KB)

wlp2s0    Link encap:以太网  硬件地址 b8:ee:65:5a:43:89  
          UP BROADCAST MULTICAST  MTU:1500  跃点数:1
          接收数据包:11151 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:12810 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000
          接收字节:10096559 (10.0 MB)  发送字节:1893779 (1.8 MB)
```
说明：

* **enp1s0** ：有线链接。此处的硬件地址为 MAC 地址，inet addr 用来表示网卡的IP地址。
* **lo** :本地回环。用来测试本地的网络，之后的127.0.0.1 代表了本机 ip 。
* **wlp2s0** :网络链接。

UP（代表网卡开启状态）RUNNING（代表网卡的网线被接上）MULTICAST（支持组播）MTU:1500（最大传输单元）
