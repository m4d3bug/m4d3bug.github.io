---
title: 每天5分钟OSP-Nova01
mathjax: true
copyright: true
comment: true
date: 2022-10-30 08:47:12
updated: 2022-10-30 23:14:37
categories:
- "Ops "
tags:
- "Linux "
- "Compute "
- "OpenStack "
- "Nova "
- "每天五分鐘玩轉OpenStack "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221030084802.png" width=50% /></center>

## 0x00️⃣前言

本文记录一些个人的nova架构的学习笔记。

<!-- more -->

## 0x01️⃣从OpenStack的架构谈起

**以下是OpenStack的架构：**



![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221030085206.png)

围绕在VM的四周，Nova 提供部署，Neutron提供网络，Glance提供镜像，cinder提供volumes。

## 0x02️⃣更近一步

nova架构

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221030105555.png)

![Nova 架构](https://www.xjimmy.com/wp-content/uploads/image/20180108/1515342369642394.png)

- API
  - nova-api：接收和相应api调用，兼容调用EC2 API的客户端工具。
- Compute Core
  - nova-scheduler：虚拟机调度服务。
  - nova-conductor：虚拟机更新数据库的前置服务。
  - nova-compute：虚拟机管理核心服务，调度Hypervisor api。
    - Hypervisor有KVM/Xen/VMWare
- Console Interface
  - nova-console：用户可以通过多种方式访问虚机的控制台： 
    - nova-novncproxy：基于 Web 浏览器的VNC 访问
    - nova-spicehtml5proxy：基于HTML5 浏览器的 SPICE 访问
    - nova-xvpnvncproxy：基于 Java 客户端的 VNC 访问
  - nova-consoleauth：负责对访问虚机控制台请求提供 Token 认证
  - nova-cert：提供 x509 证书支持
-  Database
  - Compute Database
- Message Queue
  - RabbitMQ 是nova组件通讯的关键。

## 0x03️⃣结语

今天的五分钟就到这了，期待后面的nova章节，最近刚参加过nova的debug workshop，受益匪浅。

