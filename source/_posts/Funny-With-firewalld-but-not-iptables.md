---
title: 關於firewalld與iptables趣事
mathjax: true
copyright: true
comment: true
date: 2020-06-25 23:21:37
categories:
- "Ops "
tags:
- "Linux "
- "Networking "
- "Firewalld "
- "Iptables "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/firewall.jpg" width=50% /></center>

<!-- more -->

## 事情起因

---

遇到一個**非常重要的人**問我不得不答的以下兩個firewalld的問題。

1. 讓某一些端口只能被某一些IP地址所訪問。
2. 在達成1的同時，除1以外的任何端口外不做任何限制，均可被所有IP地址訪問。
3. iptables -A INPUT -i lo -j ACCEPT的影響是什麽

## 事情經過

### firewalld部分

問題1和2咋一看，似乎應該選擇trusted zone，firewalld裏唯一一個默認放行所有端口的zone。但實驗下來，trusted zone下并不能再做黑名單的過濾，遂放棄。（如有錯誤，請斧正）

因此解決問題的思路轉爲，使用public zone，然後通過端口添加，放行全部協議，然後在rich rule中增添端口限制。

```bash
firewall-cmd --permanent --zone=public --add-port=1-65535/tcp
firewall-cmd --permanent --zone=public --add-port=1-65535/udp
firewall-cmd --reload
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source NOT address="192.168.99.99" port port="22" protocol="tcp" reject"
firewall-cmd --reload
```

### iptables部分

該命令配置的是loopback端口，作用於127.0.0.1環迴地址的規則，正常情況下與其他的iptables規則互不影響，於是可以通過以下方法來更直觀地瞭解這個情況。

```bash
[root@localhost ~]# iptables -nvL
Chain INPUT (policy ACCEPT 1021 packets, 124K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
   34  4493 ACCEPT     tcp  --  *      *       192.168.99.99        0.0.0.0/0            tcp dpt:22
   20  1200 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22
    0     0 REJECT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 521 packets, 51006 bytes)
 pkts bytes target     prot opt in     out     source               destination  
[root@localhost ~]# ssh root@127.0.0.1                                              <------目的地址使用127.0.0.1
The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:cEB3EJMdhfTrMJhyP8b3jMgv1jLzR42Y45N6SsCLmes.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '127.0.0.1' (ECDSA) to the list of known hosts.
root@127.0.0.1's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Jun 23 22:31:35 2020 from ::1
[root@localhost ~]# iptables -nvL
Chain INPUT (policy ACCEPT 1025 packets, 125K bytes)
 pkts bytes target     prot opt in     out     source               destination         
   47  7746 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0       <------可以看到流經情況    
   34  4493 ACCEPT     tcp  --  *      *       192.168.99.99        0.0.0.0/0            tcp dpt:22
   20  1200 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22
    0     0 REJECT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 569 packets, 58858 bytes)
 pkts bytes target     prot opt in     out     source               destination  
[root@localhost ~]# curl http://127.0.0.1                                           <-----觸發x2
curl: (7) Failed to connect to 127.0.0.1 port 80: Connection refused
[root@localhost ~]# iptables -nvL
Chain INPUT (policy ACCEPT 1094 packets, 135K bytes)
 pkts bytes target     prot opt in     out     source               destination         
  176 18806 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0       <------成功x2   
   34  4493 ACCEPT     tcp  --  *      *       192.168.99.99        0.0.0.0/0            tcp dpt:22
   20  1200 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22
    0     0 REJECT     tcp  --  *      *       192.168.99.4         0.0.0.0/0            tcp dpt:22 reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 712 packets, 71120 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

## 結語

firewalld還是蠻好用的，有時候想東西得反過來想。
