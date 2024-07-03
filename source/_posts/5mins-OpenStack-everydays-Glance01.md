---
title: 每天5分钟OSP-Glance01
mathjax: true
copyright: true
comment: true
date: 2022-10-28 23:14:37
updated: 2022-10-28 23:14:37
categories:
- "Ops "
tags:
- "Linux "
- "Storage "
- "OpenStack "
- "Glance "
- "每天五分鐘玩轉OpenStack "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221029000932.png" width=50% /></center>

## 0x00️⃣前言

本文记录一些个人的glance学习笔记。

<!-- more -->

## 0x01️⃣理解Image

-  安装系统，from CD or Ghost

- 效率低

- 工作量大

- 手动配置

- 备份恢复不灵活 

-  Image，模板，os+software，批量分发，快照

## 0x02️⃣理解Image Service

- 管理Image，集发现、获取和保存

- 具体功能

- REST API 查询+获取image的元数据和image本身

- 多种方式存储，文件系统、Swift、Amazon S3

- 对Instance Snapshot创建新的image

## 0x03️⃣Glance架构

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221012181155.png)

- glance-api

  - 后台进程，提供API，相应image查询、存取image的metada（丢给glance-registry）、存储image（丢给store backend）的调用。

- glance-registry

  - 主要处理image的metada，image的大小和类型。

  - 镜像类型如下：

    ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221028214306.png)

- database

  - 默认使用mysql，但redhat openstack没有.

    ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221028230918.png)

- store backend，支持多种， /etc/glance/glance-api.conf 配置，不同配置[遵循](http://docs.openstack.org/liberty/config-reference/content/configuring-image-service-backends.html)

  - A directory on a local file system（这是默认配置），uuid名的image存储在对应文件夹

    ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221028231400.png)

    ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221028231302.png)

    ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221028231316.png)

  - GridFS

  - Ceph RBD

  - Amazon S3

  - Sheepdog

  - OpenStack Block Storage (Cinder)

  - OpenStack Object Storage (Swift)

  - VMware ESX

## 0x04️⃣总结感悟

简洁明了了解一下glance，接下来实操glance。
