---
title: 搭建Wireguard主备切换架构
mathjax: true
copyright: true
comment: true
date: 2023-08-27 13:41:46
updated: 2023-08-27 13:41:33
categories:
- "Ops "
tags:
- "Linux "
- "Wireguard "
- "Network "
- "UDP "
- ”Syncthing “
- "CloudFlare "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/wireguard.png" width=50% /></center>

## 0x00️⃣ 前言

本文旨在阐述个人搭建wireguard主备切换架构的细节。[wireguard的相关搭建](https://blog.madebug.net/Ops/2021-07-04-Use-Wireguard-to-rebuild-a-intranet-in-China.html)不做赘述，主要解决单server没有冗余的问题。

<!-- more -->

组成技术栈如下：

- 服务器主体搭建，wg-gen-web
- 配置文件同步，syncthing
- 主备DNS解析切换，cloudflare

## 0x01️⃣Wireguard

和之前的配置略微有些不同，json api监听在0.0.0.0，便于cloudflare监控异常。完整搭建在[这里](https://blog.madebug.net/Ops/2021-07-04-Use-Wireguard-to-rebuild-a-intranet-in-China.html)，主备都需要操作。

- wg-1

  ~~~
  version: '3.6'
  services:
    wg-gen-web:
      image: vx3r/wg-gen-web:latest
      container_name: wg-gen-web
      restart: always
      expose:
        - "8080/tcp"
      ports:
        - 172.24.0.1:80:8080
      environment:
        - WG_CONF_DIR=/data
        - WG_INTERFACE_NAME=wg0.conf
        - OAUTH2_PROVIDER_NAME=fake
        - WG_STATS_API=http://172.17.0.1:8182
      volumes:
        - /etc/wireguard:/data
      network_mode: bridge
    wg-json-api:
      image: james/wg-api:latest
      container_name: wg-json-api
      restart: always
      cap_add:
        - NET_ADMIN
      network_mode: "host"
      command: wg-api --device wg0 --listen 0.0.0.0:8182
  ~~~

- wg-2

  ~~~
  version: '3.6'
  services:
    wg-gen-web:
      image: vx3r/wg-gen-web:latest
      container_name: wg-gen-web
      restart: always
      expose:
        - "8080/tcp"
      ports:
        - 172.24.0.2:80:8080
      environment:
        - WG_CONF_DIR=/data
        - WG_INTERFACE_NAME=wg0.conf
        - OAUTH2_PROVIDER_NAME=fake
        - WG_STATS_API=http://172.17.0.1:8182
      volumes:
        - /etc/wireguard:/data
      network_mode: bridge
    wg-json-api:
      image: james/wg-api:latest
      container_name: wg-json-api
      restart: always
      cap_add:
        - NET_ADMIN
      network_mode: "host"
      command: wg-api --device wg0 --listen 0.0.0.0:8182
  ~~~

  

## 0x02️⃣Syncthing

然后配置Syncthing用于同步前面生成的wireguard配置文件。

- wg-1 

  ~~~
  version: "3"
  services:
    syncthing:
      image: syncthing/syncthing
      container_name: syncthing
      hostname: 1-syncthing
      environment:
        - PUID=1000
        - PGID=1000
      volumes:
        - /etc/wireguard:/wireguard
      ports:
        -  172.24.0.1:8384:8384 # Web UI
        - 22000:22000/tcp # TCP file transfers
        - 22000:22000/udp # QUIC file transfers
        - 21027:21027/udp # Receive local discovery broadcasts
      restart: unless-stopped
  ~~~

- wg-2

  ~~~
  version: "3"
  services:
    syncthing:
      image: syncthing/syncthing
      container_name: syncthing
      hostname: 2-syncthing
      environment:
        - PUID=1000
        - PGID=1000
      volumes:
        - /etc/wireguard:/wireguard
      ports:
        - 172.24.0.2:8384:8384 # Web UI
        - 22000:22000/tcp # TCP file transfers
        - 22000:22000/udp # QUIC file transfers
        - 21027:21027/udp # Receive local discovery broadcasts
      restart: unless-stopped
  ~~~

- 然后网页配置它们的同步策略和互连。鼠标点点就行，没什么技术含量。

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/d3a70cdd417cc494c506764912f9f5b.png)

## 0x03️⃣cloudflare

创建load balancing，在Traffic下，依次创建Manage Monitors，Manage Pools 和 Load Balancer。

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828160624.png)

- Manage Monitors

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828161815.png)

- Manage Pool

  先创建pool，然后把服务器信息填入，一个pool一台服务器，因为wireguard的模式只能允许主备方式运行。而非常规的http可以提供响应检测搭配weight来进行工作。因此我们需要设置两个pool，其中一个pool作为failback选项。

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828161555.png)

- Load Balancer

  我们这里的目的是想要负载均衡，在其中一个服务器宕机的时候切换主备，而wireguard的endpoint依旧使用域名解析，因此这里的Hostname需要设入对应的域名，并且关闭cdn。因为wireguard协议不能经过cdn转发。

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828161038.png)

  

  加入对应的pool，fallback pool选择备用的ip。

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828162002.png)

重点设置failover这里

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828162101.png)

## 0x04️⃣测试

对其中一台的grub参数修改，确保其延期重启超过60秒切换的阈值以达到模拟效果。

~~~
# cat /etc/default/grub
GRUB_TIMEOUT=120
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/almalinux-swap rd.lvm.lv=almalinux/root rd.lvm.lv=almalinux/swap"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true

# grub2-mkconfig -o /boot/grub2/grub.cfg
# reboot
~~~

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20230828163204.png)

最后顺利完成主备切换做到自愈。

## 0x05️⃣ 总结

最开始的需求是wireguard希望能有ha的工作模式，因此有找到[netmaker](https://itnext.io/how-to-deploy-a-highly-available-wireguard-network-management-server-on-kubernetes-294e23c7abcb)这一个方案。但是显然netmaker最大的问题还是强行运行443端口，也不太确定能否支持cloudflare-tunnel作为ingress，因此放弃。然后找到一个[基于vrrp搭建](https://noteblok.net/2022/07/28/a-highly-available-wireguard-vpn-setup/)的高可用方案。但公网运行vrrp协议需要qugga，而这个在CentOS 8已经废弃，因此也不适合我在公网搭建的场景。最后了解到cloudflare有fallback dns的功能，$5/月，完美解决。
