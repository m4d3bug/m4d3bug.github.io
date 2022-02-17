---
title: 从user.conf谈起
mathjax: true
copyright: true
comment: true
date: 2022-02-16 20:41:17
categories: "Ops "
tags: 
- "Linux "
- "ulimit "
- "Services "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/ulimit-command.jpg" width=50% /></center>

## 0x00 前言
最近遇到一个问题，`/etc/system/system.conf` 和`/etc/system/user.conf` 的区别，没看到有很合适的笔记，简单记录一下。
<!-- more -->

## 0x01 概念
遇到不懂的Linux概念，自然而言首先求助于`Linux`自带的文档。

```bash
# rpm -qd systemd |grep user
/usr/share/man/man5/systemd-user.conf.5.gz
/usr/share/man/man5/sysusers.d.5.gz
/usr/share/man/man5/user.conf.d.5.gz
/usr/share/man/man8/systemd-sysusers.8.gz
/usr/share/man/man8/systemd-sysusers.service.8.gz
/usr/share/man/man8/systemd-user-sessions.8.gz
/usr/share/man/man8/systemd-user-sessions.service.8.gz

# rpm -qd systemd |grep systemd-system
/usr/share/man/man5/systemd-system.conf.5.gz
/usr/share/man/man8/systemd-system-update-generator.8.gz

# man 5 /usr/share/man/man5/systemd-system.conf.5.gz
# man 5 /usr/share/man/man5/systemd-user.conf.5.gz

DESCRIPTION
When run as a system instance, systemd interprets the configuration file system.conf and the files in system.conf.d directories; when run as a user instance, systemd interprets the configuration file user.conf and the files in user.conf.d directories. These configuration files contain a few settings controlling basic manager operations. See systemd.syntax(5) for a general description of the syntax.
```
至此，我们可以得到一个简单清晰的关系链，当作为`system instance`时，`systemd`会引入`system.conf`，当作为`user instance`时，`systemd`会引入`user.conf`。但是新的问题又来了。

## 0x02 user instance
简单`google`整理一下思路。

* 概念

```bash
和登陆行为的强关联
The systemd user instance is started after the first login of a user and killed after the last session of the user is closed. Sometimes it may be useful to start it right after boot, and keep the systemd user instance running after the last session closes,

systemd offers users the ability to manage services under the user’s control with a per-user systemd instance, enabling users to start, stop, enable, and disable their own units.
https://wiki.archlinux.org/title/Systemd/User
```
* [样例](https://www.unixsysadmin.com/systemd-user-services/)

PS:  我的过程中加入了`export XDG_RUNTIME_DIR=/run/user/$(id -u)`, [该变量用于设置用户自动登录](https://askubuntu.com/questions/872792/what-is-xdg-runtime-dir)。

```bash
[root@rhel8 ]# useradd -d /home/myapp -m -s /bin/bash -c "My application account" myapp
[root@rhel8 ]# loginctl enable-linger myapp
[root@rhel8 ]# 
[root@rhel8 ]# su - myapp
[myapp@rhel8 ~]$ mkdir -p ~/.config/systemd/user/
[myapp@rhel8 ~]$ cat >> ~/.config/systemd/user/myapp.service << EOF
[Unit]
Description=My demo application

[Service]
ExecStart=/usr/bin/python3 -m http.server 8080
WorkingDirectory=/home/myapp/html
EOF
[myapp@rhel8 ~]$ mkdir /home/myapp/html
[myapp@rhel8 ~]$ echo "Hello World" > /home/myapp/html/index.html
[myapp@rhel8 ~]$ export XDG_RUNTIME_DIR=/run/user/$(id -u)
[myapp@rhel8 ~]$ systemctl --user daemon-reload
[myapp@rhel8 ~]$ systemctl --user status myapp.service
● myapp.service - My demo application
   Loaded: loaded (/home/myapp/.config/systemd/user/myapp.service; static; vendor preset: enabled)
   Active: inactive (dead)
[myapp@rhel8 ~]$ systemctl --user start myapp.service
[myapp@rhel8 ~]$ systemctl --user status myapp.service
● myapp.service - My demo application
   Loaded: loaded (/home/myapp/.config/systemd/user/myapp.service; static; vendor preset: enabled)
   Active: active (running) since Tue 2020-05-26 12:28:04 BST; 1s ago
 Main PID: 1169 (python3)
   CGroup: /user.slice/user-1001.slice/user@1001.service/myapp.service
           └─1169 /usr/bin/python3 -m http.server 8080

[myapp@localhost ~]$ cat /proc/1169/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        unlimited            unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             14603                14603                processes
Max open files            1024                 262144               files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       14603                14603                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```
至此，`user instance`和`user.conf`都了解得七七八八了，那就检验一下吧。

## 0x03 检验
检验的方式和`system.conf`一样，在`/etc/systemd/user.conf`中设置。

```bash
[root@rhel8 ]# cat  /etc/systemd/user.conf  |grep -vE '^$|^#'
[Manager]
DefaultLimitNOFILE=65535

[root@rhel8 ] # systemctl daemon-reexec
[myapp@rhel8 ~]$ export XDG_RUNTIME_DIR=/run/user/$(id -u)
[myapp@rhel8 ~]$ systemctl daemon-reexec --user
[myapp@rhel8 ~]$ systemctl --user status myapp
● myapp.service - My demo application
   Loaded: loaded (/home/myapp/.config/systemd/user/myapp.service; static; vendor preset: enabled)
   Active: active (running) since Wed 2022-02-16 04:27:33 PST; 5min ago
 Main PID: 3943 (python3)
   CGroup: /user.slice/user-1001.slice/user@1001.service/myapp.service
           └─3943 /usr/bin/python3 -m http.server 8080
```
不一会儿

```bash
[myapp@rhel8 ~]$ cat /proc/3943/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        unlimited            unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             14603                14603                processes
Max open files            65535                65535                files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       14603                14603                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```
## 0x04 总结
本文简单地从`user.conf`谈起，到`user instance`的探讨，最后到检验它们两者的关联。可以看到，时至今日，systemd仍然在进化，而多用户管理场景，在云计算不合适的场景下，也是复用资源的一种思路。



