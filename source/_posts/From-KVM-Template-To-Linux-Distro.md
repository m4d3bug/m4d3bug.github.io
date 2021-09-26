---
title: 從KVM模板到OS淺談
mathjax: true
copyright: true
comment: true
date: 2021-09-26 21:38:05
categories:
- "Ops "
tags:
- "Linux "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/Linux-distro.png" width=50% /></center>

本文旨在阐述如何使用virt-sysprep制作kvm的qcow2格式的模板，以及一些感想。

<!-- more -->

## 0x00 前言

许久没有更新博客，正在花时间将手上的几台HomeLab机器整合划分不同的域，写blog素材upup，而kvm的各项工具正在蓬勃发展，现已不再需要osp和rhev也能够获得比较良好的虚拟机管理体验了。

## 0x01 工具介绍

这次的主角是virt-sysprep，以下方法适用于制作Ubuntu1804，Ubuntu2004，CentOS7，Centos Stream 8的模板。

## 0x02 制作过程

### 新建模板

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/ubuntu-template.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/ubuntu-template.png)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/centos7-template.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/centos7-template.png)

- 使用lvm，并且设置确保/home目录没有使用
- 设置普通用户并赋予sudo权限
- 选择live-server

### 清除标记

安装完成后使用virt-sysprep清除以下

- 默认的日志项
- 重置ssh相关
- 清除machine-id
- 时区设定

```bash
# virt-sysprep -d ubuntu2004_server --timezone 'Asia/Shanghai' --operations machine-id --no-logfile -v -x
# virt-sysprep -d centos7_server --timezone 'Asia/Shanghai' --operations machine-id --no-logfile -v -x
```

## 0x03 不同发行版

最近有不少不同Linux发行版的各种新闻，CentOS Stream 8取代CentOS 8，国家机关强推统信、麒麟OS等各种关乎不同OS的新闻出现，趁这篇博文撰写的时候谈一下自己粗浅的见解。

1. 谁最好 

我认为没有最好的Linux发行版，只有最适合自己的发行版。每种发行版于我而言，它们所做的一些开箱即用的优化以及一些特性都让我爱不释手。

    - Fedora：rpm系的最上游，是成熟的kvm管理工具cockpit-kvm的最优先适配。rpm系breack change的试验田。
    - CentOS Stream：Stream的出现，我认为顺应了云原生浪潮。小版本的取消，使得CentOS上的创新能够以最低、最快的门槛适配RHEL。CentOS过去的角色，Rocky, Almalinux、TencentOS这些发行版可以轻松填补，但是CentOS Stream的角色它们却不能填补。这对开源软件界进入企业流行无疑是一大利好。
    - RHEL: 集最多硬件支持、最优开源软件编译参数实践、最优商业支持Linux于一身。
    - Rocky，Almalinux、统信UOS服务器版、TencentOS： 现今的CentOS角色，并且也推出了[便宜的支持策略](https://www.zdnet.com/article/centos-clone-rocky-linux-gets-technical-support/)，像Oracle linux一样，更低的商业成本享受rpm系。
    - Debian：apt系的最上游，拥有比RHEL更“自由”的发行版，提供了进入Linux流行的更低门槛。毫无疑问会被更多不旨在进入企业流行盈利的软件爱好者所喜爱，须知也不是所有的开源应用都热衷于进入到企业盈利。
    - Ubuntu：apt系的扛把子，漂亮优雅的桌面风格，更新的内核。现在也采取了和RHEL一样的“十年”支持。额外使用了cloud-init来实现更易分发的特性，但是这也带来了一定侵入性，可选的apt系商业支持。
    - Kali、deepin、统信UOS桌面版: 拥有很多适合特定场景的软件集合，提供开箱即用的桌面大杂烩工具集。
    - openEular：结合了Ubuntu发行策略优势，以自身小版本为上游的rpm系产物。一方面背靠rpm系可以获得成熟的企业支持经验和古稀硬件的支持经验，一方面采用更新内核，可以更好地衔接上游的开源成果。也隐去了人们对CentOS Stream的担忧。
    - RHCOS、RancherOS、Photon OS：专为容器而生的OS，从机器的角度提供更加可控的K8S运行时的OS，但从人类的角度更加可控性更差。它们应用在物理机上能节省更多的资源给上层，但也意味着更多的开源特性依赖厂商支持。
    - 其他：没有体验不做评价。
2. 选择原则
    1. 商业成本&业务中断容忍度（高？中？低？）
    2. 安全要求（高？中？低？）
    3. 虚拟化程度？（高？中？低？容器化？虚拟机？物理机？）
    4. 技术栈支持度（高？中？低？内核依赖？组件依赖？特性依赖？）
3. 个人癖好
    1. 当无可避免地需要运行不可信代码时，会优先考虑拥有selinux的rpm系。 
    2. 当需要开箱即用的软件环境时，会优先根据其支持程度选择apt系。
    3. 当有稀奇古怪的硬件时，会优先考虑rpm系。 
    4. 当容器化程度很高，底层os需要及时更新及时，会考虑CentOS Stream、openEular这种不以牺牲人类可控性但又不至于更新太慢的OS。

## 0x04 总结

- virt-sysprep挺好用的，cloud-init拜拜。
- 各大发行版都有自己的使用场景，应结合使用场景，相互借鉴，大家好才是真的好。“无他，唯手熟尔”。
- 有机会搞一个混杂各种发行版的K8S。
- 能够在这么纷繁复杂的世界中诞生Linux，已经是很了不起的事情了。
