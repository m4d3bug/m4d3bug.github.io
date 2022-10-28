---
title: 使用headscale搭建國内局域網
mathjax: true
copyright: true
comment: true
date: 2022-10-16 21:24:06
updated: 2022-10-16 21:24:06
categories:
- "Ops "
tags:
- "Linux "
- "Headscale "
- "Network "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221016212516.png" width=50% /></center>

## 0x00 前言

什么是headscae ？

<!-- more -->

> *An open source, self-hosted implementation of the Tailscale control server.*

什么是Tailscale ？

> *Tailscale is [a modern VPN](https://tailscale.com/) built on top of [Wireguard](https://www.wireguard.com/). It [works like an overlay network](https://tailscale.com/blog/how-tailscale-works/) between the computers of your networks - using [NAT traversal](https://tailscale.com/blog/how-nat-traversal-works/).*
>
> *Everything in Tailscale is Open Source, except the GUI clients for proprietary OS (Windows and macOS/iOS), and the control server.*
>
> *The control server works as an exchange point of Wireguard public keys for the nodes in the Tailscale network. It assigns the IP addresses of the clients, creates the boundaries between each user, enables sharing machines between users, and exposes the advertised routes of your nodes.*
>
> *A [Tailscale network (tailnet)](https://tailscale.com/kb/1136/tailnet/) is private network which Tailscale assigns to a user in terms of private users or an organisation.*

之前使用wireguard和zerotier（甚至还试了docker版的zerotier root模式）搭建了局域网，现在用headscale再额外搭一个网。有人会嫌网络多吗？反正我不会。我对搭局域网的打洞软件要求不外乎是那么几点：

- 可迁移
- 不跨境
- 最好有面板

## 0x01 前置准备

- docker
- docker-compose
- [cloudflare tunnel](https://www.cloudflare.com/zh-cn/products/tunnel/) (自行注册，需要绑定支付方式)
- 域名托管在cloudflare 

## 0x02 headscale服务端

- **headscale部分**

~~~shell
$ mkdir -p /opt/containers/headscale/container-config
$ cd /opt/containers/headscale/container-config
$ wget -O ./config.yaml https://img.madebug.net/juanfont/headscale/main/config-example.yaml
$ sed -i '13c server_url\: https\:\/\/<your-domain>' config.yaml #加入时的回显内容，可选项
$ sed -i '58c \ \ \-\ 10.9.9.0/24' config.yaml #自定义网段，可选项
$ sed -i '214c \ \ \magic_dns:\ false' config.yaml
$ cd ..
$ cat > docker-compose.yaml << EOF
services:
  headscale:
    container_name: headscale
    #    image: headscale/headscale:latest-alpine 
    image: headscale/headscale:0.16.4 #最新版对cloudflare的websocket支持似乎有问题 
    restart: unless-stopped
    # ports:
      # - 8080:8080
    volumes:
      - ./container-config:/etc/headscale
      - ./container-data:/var/lib/headscale
    entrypoint: headscale serve
    networks:
      reverseproxy-nw:

  headscale-ui:
    container_name: headscale-ui
    # image: ghcr.io/gurucomputing/headscale-ui:latest 
    image: madebug/ghcr.io.gurucomputing.headscale-ui:latest #sync了一个在dockerhub
    restart: unless-stopped
    networks:
      reverseproxy-nw:

networks:
  reverseproxy-nw:
    external: true
EOF
$ docker network create reverseproxy-nw
$ docker-compose up -d 
$ curl http://0.0.0.0:8080/windows
$ curl http://0.0.0.0:9090/metrics
~~~

- **cloudflare tunnel部分**

  ​         众所周知，国内的vps的80和443都是默认不启用的，这里用cloudflare tunnel绕过80和443的限制，并且转发headscale-ui和headscale的服务达到国内vps就可以搭建的目的。

  ​        [打开cloudflare zero trust的界面](https://dash.teams.cloudflare.com/)，Access >> Tunnels >> Create a tunnel >> Tunnel name(随便填) >> Save tunnel >> Docker

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221018183537.png)

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221018184140.png)

  ​      在完成上述cloudflare tunnel侧的设置，就可以回到vps，尝试进行cloudflared的第一次创建和连接。注意加入`--network reverseproxy-nw`

  ~~~shell
  $ docker run --network reverseproxy-nw cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoiY2I5YzllOGY0ZjE3ZjFmOGU2NmIyOWU5N2M3NTIxNjIiLCJ0IjoiNTQ0OTFiY2ItMjMwMS00ZTI5LTk3OTgtN2E0OWI1Yzk5ZjNmIiwicyI6Ik1HTTRaVEkwTW1RdFlqWmxNaTAwT0RrekxUbGhPVFF0WWpBNFpUQTRNemMwT0RReSJ9
  2022-10-18T10:59:08Z INF Thank you for trying Cloudflare Tunnel. Doing so, without a Cloudflare account, is a quick way to experiment and try it out. However, be aware that these account-less Tunnels have no uptime guarantee. If you intend to use Tunnels in production you should use a pre-created named tunnel by following: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps
  2022-10-18T10:59:08Z INF Requesting new quick Tunnel on trycloudflare.com...
  2022-10-18T10:59:11Z INF +--------------------------------------------------------------------------------------------+
  2022-10-18T10:59:11Z INF |  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
  2022-10-18T10:59:11Z INF |  https://bars-bidding-tucson-transportation.trycloudflare.com                              |
  2022-10-18T10:59:11Z INF +--------------------------------------------------------------------------------------------+
  ...
  
  ~~~

  ​        进展顺利，你就可以看到connectors显示以下信息了

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221018190900.png)

  ​        点击next进行headscale的设置：

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221018191216.png)

  ​        headscale-ui如法炮制，注意路径，以及关闭tls验证。

  ![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221018191845.png)

  ​        测试成功，你应该可以在这个tunnel下的Public Hostname的位置看到两条转发规则，并且它们都是工作的。就可以加入到docker-compose.yaml了，它看起来应该是这样的。

  ~~~shell
  services:
    headscale:
      container_name: headscale
      #    image: headscale/headscale:latest-alpine
      image: headscale/headscale:0.16.4
      restart: unless-stopped
      # ports:
        # - 8080:8080
      volumes:
        - ./container-config:/etc/headscale
        - ./container-data:/var/lib/headscale
      entrypoint: headscale serve
      networks:
        reverseproxy-nw:
  
    headscale-ui:
      container_name: headscale-ui
      image: ghcr.io/gurucomputing/headscale-ui:latest
      restart: unless-stopped
      networks:
        reverseproxy-nw:
  
    cloudflared:
      container_name: cloudflared
      image: cloudflare/cloudflared:latest
      entrypoint: ["cloudflared", "tunnel", "--no-autoupdate", "run", "--token", "eyJhIjoiY2I5YzllOGY0ZjE3ZjFmOGU2NmIyOWU5N2M3NTIxNjIiLCJ0IjoiNTQ0OTFiY2ItMjMwMS00ZTI5LTk3OTgtN2E0OWI1Yzk5ZjNmIiwicyI6Ik1HTTRaVEkwTW1RdFlqWmxNaTAwT0RrekxUbGhPVFF0WWpBNFpUQTRNemMwT0RReSJ9"]
      networks:
        reverseproxy-nw:
  
  networks:
    reverseproxy-nw:
      external: true
  ~~~

  没有问题直接`docker-compose up -d`

- **headscale-ui部分**

  前面基本完成了90%的工作了，接下来就是登录headscale-ui进行一些登录的设置了。

1.  headscale生成页面登录的token

   ~~~
   $ docker exec headscale headscale apikeys create
   ~~~

2.  访问https://<前面设定的域名>/web，然后填入token即可，点击`Test Server Settings`。

3.  创建user就是创建namespace，用来隔离多个不同网段。

## 0x03 headscale客户端

1. 安装[tailscale客户端](https://pkgs.tailscale.com/stable/)。

2. 在headscale-ui创建**Preauth Keys**。

3. 然后使用以下命令加入。

   ```
   $ tailscale up --login-server https://<你前面的域名> --accept-routes=true --accept-dns=false --authkey <你自己设的Preauth Keys>
   ```

4. Windows 加入方法。

   ```
   https://<你前面的域名>/windows
   ```

## 0x04 结语

- headscale的打洞能力超强，并且自身还可以搭配/作为derper（derper类似zerotier的moon）。

- headscale本身只是实现了服务端，客户端可以直接沿用tailscale进行加入。

- 迁移的时候除了`containers-data`下的`private.key`不需要之外，其他都需要迁走。

- 我们的方案用了cloudflare tunnel，当然可以[优选出优质ip](https://github.com/XIU2/CloudflareSpeedTest)，然后在软路由侧写入解析。

- 还是很建议[单独搭一个derper的](https://icloudnative.io/posts/custom-derp-servers/#%E4%BD%BF%E7%94%A8%E7%BA%AF-ip)，未必要把官方的derper全部去掉，但是这打洞效率很高。

- [headscale支持自定义网段转发](https://tailscale.com/kb/1019/subnets/?tab=linux#enable-ip-forwarding)。

   ```shell
   $ tailscale up --login-server https://<你前面的域名> --accept-routes=true --accept-dns=false --authkey <你自己设的Preauth Keys> --advertise-routes=x.x.x.x/xx,x.x.x.x/xx
   ```

## 0x05 鸣谢

- [Setting Up Headscale :: Guru Computing Blog](https://blog.gurucomputing.com.au/smart-vpns-with-headscale/setting-up-headscale/)
- [Tailscale 基础教程：Headscale 的部署方法和使用教程 – 云原生实验室 - Kubernetes|Docker|Istio|Envoy|Hugo|Golang|云原生](https://icloudnative.io/posts/how-to-set-up-or-migrate-headscale/)

