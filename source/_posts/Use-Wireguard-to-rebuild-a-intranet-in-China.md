title: 使用Wireguard基於國内網絡異地組網實踐
mathjax: true
copyright: true
comment: true
date: 2021-07-04 20:11:46
updated: 2021-07-05 07:33:33
categories:
- "Ops "
tags:
- "Linux "
- "Wireguard "
- "Network "
- "UDP "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/wireguard.png" width=50% /></center>

本文旨在阐述个人使用wireguard基于国内网络异地组网实践的细节。

<!-- more -->

## 0x00 需求分析

需求有哪些：

- 异地家庭带宽下组网。
- 运行环境繁多，包括但不仅限于，Android、IOS、软路由、Windows、Linux、MacOS、K8s等。
- 延迟低于100ms。
- 客户端稳定，维护简单，便于后续更新迭代。
- 经济实惠。
- 高可用，便于切换、切换顺滑。
- 不过度依赖国内服务。
- 安全可靠。

DDNS，国内挂靠域名。公网IP，需要耗费防护精力。Zerotier，客户端不稳定。Tailscale，客户端时延过大。n2n，未深入。frp，基于端口不利于过多进程场景，等等等等……

Wireguard，被Linus盛赞的艺术，前几年刚出的时候接触过，印象颇为深刻。这几年来被身边各种各样的资讯推荐，youtube，公众号。似乎它已经足够成熟了，于是有了本文。

## 0x01 服务端

服务端主要负责以下几项工作：

- 自动生成配置文件
- 自动加载配置文件
- 转发流量

演示使用Ubuntu 20.04 TLS，其他发行版请参照[这里](https://fuckcloudnative.io/posts/wireguard-docs-practice/)。

### 软件安装

```bash
# apt install wireguard wireguard-tools  byobu docker.io docker-compose -y
```

### 系统设置

```bash
$ 内核转发
# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-sysctl.conf
# echo "net.ipv4.conf.all.proxy_arp = 1" >> /etc/sysctl.d/99-sysctl.conf
# sysctl -p /etc/sysctl.d/99-sysctl.conf

$ iptables配置端口转发自定义网段
# iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -i wg0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT
# iptables -t nat -A POSTROUTING -s 10.9.8.0/24 -o eth0 -j MASQUERADE

$ 初始化wireguard
# wg-quick up wg0
# wg-quick down wg0
```

### 托管设置

```bash
$ 直接使用wg-gen-web管理, 容器运行填入docker0的IP
# nano docker-compose.yaml
version: '3.6'
services:
  wg-gen-web:
    image: vx3r/wg-gen-web:latest
    container_name: wg-gen-web
    restart: always
    expose:
      - "8080/tcp"
    ports:
      - 8888:8080
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
    command: wg-api --device wg0 --listen 172.17.0.1:8182

# docker-compose up -d
```

### 软件设置

web访问<web_ip:8888>进行配置

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/wireguardserver.png?raw=ture](https://img.madebug.net/m4d3bug/images-of-website/master/blog/wireguardserver.png?raw=ture)

```bash
$ 检查生成的配置文件
# nano /etc/wireguard/wg0.conf

# Updated: 2021-06-14 09:51:23.220280864 +0000 UTC / Created: 2021-06-12 18:27:38.627308446 +0000 UTC
[Interface]
Address = 10.9.8.1/24
ListenPort = 9999
PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

PreUp = echo WireGuard PreUp
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PreDown = echo WireGuard PreDown
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

### 自动化管理

自动加载配置文件、自动生成配置文件。

```bash
$ 加入ExecReload在不中断活跃连接的情况下重新加载配置文件
# nano /usr/lib/systemd/system/wg-quick@.service
[Unit]
Description=WireGuard via wg-quick(8) for %I
After=network-online.target nss-lookup.target
Wants=network-online.target nss-lookup.target
PartOf=wg-quick.target
Documentation=man:wg-quick(8)
Documentation=man:wg(8)
Documentation=https://www.wireguard.com/
Documentation=https://www.wireguard.com/quickstart/
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/wg-quick up %i
ExecStop=/usr/bin/wg-quick down %i
ExecReload=/bin/bash -c 'exec /usr/bin/wg syncconf %i <(exec /usr/bin/wg-quick strip %i)'
Environment=WG_ENDPOINT_RESOLUTION_RETRIES=infinity

[Install]
WantedBy=multi-user.target

$ 添加自动Reload
# nano /etc/systemd/system/wg-gen-web.service
[Unit]
Description=Restart WireGuard
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl reload wg-quick@wg0.service

[Install]
WantedBy=multi-user.target

$ 添加监听路径
# nano /etc/systemd/system/wg-gen-web.path
[Unit]
Description=Watch /etc/wireguard for changes

[Path]
PathModified=/etc/wireguard

[Install]
WantedBy=multi-user.target

$ 刷新上述配置
# systemctl daemon-reload
# systemctl start wg-quick@wg0
# systemctl enable wg-gen-web.service wg-gen-web.path wg-quick@wg0 --now
```

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/wg_client_setup.png?raw=ture](https://img.madebug.net/m4d3bug/images-of-website/master/blog/wg_client_setup.png?raw=ture)

当然，另外你还会需要为这个页面做一个密码防护。[Nginx Basic Authentication]

## 0x02 客户端

### 软件安装

```bash
# apt install wireguard wireguard-tools -y
```

### 软件设置

从服务端直接下载配置文件到**/etc/wireguard/**，并命名为wg0.conf

```bash
$ 同上，略，可以自行选择是否添加对配置文件的监听
```

## 0x03 进阶

至此，你已经可以拥有一个基于wireguard搭建的中继局域网了。但我们还有部分需求没解决。

- 不过度依赖国内服务。

其实这一层的隐忧，就是vps的续费问题，可以看见上面的配置文件我使用的是中继服务器的域名进行设定，手动切换域名解析的影响对我来说是可以接受的。但还需要客户端做一个额外设定进行配合。

### 客户端额外配置

[wireguard的域名解析并不是每个请求都执行的](https://lists.zx2c4.com/pipermail/wireguard/2017-November/002028.html)。因此我们需要为客户端进程添加一个定时脚本，来帮助其定时重新解析，以便在问题发生时，我们只需要关注域名的解析情况，即可达到蓝绿切换的功能，**但需要提醒一点，这样子做的不可控点在于域名解析生效的时长，请预留足够的窗口操作期，并配合良好的监控机制**。以下例子仅包含CentOS7 和Ubuntu 20.04、[这里是Archlinux的出处](https://wiki.archlinux.org/title/WireGuard#:~:text=details-,endpoint%20with%20changing%20ip,-After)。

```bash
$ 查找wireguard-tools包含的解决脚本

# find / -name reresolve-dns.sh
Ubuntu
/usr/share/doc/wireguard-tools/examples/reresolve-dns/reresolve-dns.sh
CentOS
/usr/share/doc/wireguard-tools-1.0.20210424/contrib/reresolve-dns/reresolve-dns.sh

$ 设置三十秒的定时器
# nano /etc/systemd/system/wireguard_reresolve-dns.timer
[Unit]
Description=Periodically reresolve DNS of all WireGuard endpoints

[Timer]
OnCalendar=*:*:0/30

[Install]
WantedBy=timers.target

$ 设置相应的脚本执行服务
# nano /etc/systemd/system/wireguard_reresolve-dns.service
[Unit]
Description=Reresolve DNS of all WireGuard endpoints
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for i in /etc/wireguard/*.conf; do /usr/share/doc/wireguard-tools/examples/reresolve-dns/reresolve-dns.sh "$i"; done'

# systemctl daemon-reload
# systemctl enable wireguard_reresolve-dns.timer wireguard_reresolve-dns.service --now
# systemctl reload wg-quick@wg0
```

## 0x04 测速

完成上述设定之后，顺手做了一个国内的测速。很有幸处在UDP表现良好的地域，并且突破了良心云限定的带宽墙。(你没看错，就是用tcp_bw测，或者你喜欢可以用udp_bw看看结果)

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/speed_test_from_client.jpg?raw=true](https://img.madebug.net/m4d3bug/images-of-website/master/blog/speed_test_from_client.jpg?raw=true)

## 0x05 总结

- 后续可以玩得更花一点。
- 本文放弃了full mesh的组网方式。~~显然我比较懒。~~
- cloudflare支持dns的fallback设置，五刀一个月，可以允许单点故障时解析到另一个ip（伪高可用）。
- 有理由相信，会逐渐开放对UDP的相关QOS策略，毕竟http3 就是基于udp实现的。
- 切换服务器时需要的窗口期比较长，可以配合tcpdump在客户端抓包观察，并且不要急着销毁已经在工作的实例。
- 扩展设定未在更多客户端进行测试。~~显然我比较懒。~~
- 国内互联互通，其实是对VPN不拦截的。特征明显，但不敏感。
- wireguard发展日趋成熟稳定，某大厂已经用在公司关键网络负载了。

## 0x06 鸣谢

- [WireGuard 配置教程：使用 wg-gen-web 来管理 WireGuard 的配置 – 云原生实验室 - Kubernetes|Docker|Istio|Envoy|Hugo|Golang|云原生](https://fuckcloudnative.io/posts/configure-wireguard-using-wg-gen-web/)
- [WireGuard - ArchWiki](https://wiki.archlinux.org/title/WireGuard)
