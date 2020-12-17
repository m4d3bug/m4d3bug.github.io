---
title: sendmail的深入瞭解III
mathjax: true
copyright: true
comment: true
date: 2020-10-02 16:55:26
categories:
- "Ops "
tags:
- "sendmail "
- "IMAP "
- "POP "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/LinuxAdministrationHandbook" width=50% /></center>

本文記錄關於郵件服務器的相關。

<!-- more -->
## 0x00 基本原理

- 四個功能
    - 接收從UA來的郵件提交，並發送到相應郵件系統。
    - 接收外部傳入的郵件。
    - 郵件遞送到最終用戶的郵箱。
    - 用戶可通過IMAP或POP訪問。
- 上述功能可以是同一台機器或不同機器。

## 0x01 使用郵件服務器

- 直接接入Internet的站點。

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/DesignEmailSystem.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/DesignEmailSystem.jpg)

    - 傳入傳出需要防火牆隔離。
    - 多台傳入可搭配負載均衡。
    - 單獨的機器可充當客戶機來備份MX記錄。
    - 不建議使用NFS來共享/var/spool/mail
- 不直接接入Internet的站點。

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/DeployEmailSystemII.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/DeployEmailSystemII.jpg)

## 0x02 使用IMAP或POP

- IMAP和POP是用户的桌面机器连接到网络中时用来下载电子邮件的协议。
- 使用它们的时候要求有口令，确保使用一个采用SSL加密的协议版本（IMAPS和POPS）。
- IMAP可以在多个站点之间管理邮件文件夹。
- POP（Post Office Protocol，邮局协议）与IMAP相似，但它假定了一个模型，在这个模型中所有邮件都是从服务器下载到PC的。这些邮件可以从服务器上删除（在此情况下就不能备份了）或者保存在服务器上（在此情况下您的邮件缓冲区文件就会变得越来越大）

> 如果用户从来都不删除任何邮件消息的话，这两种协议都可以变成消耗资源的大户。在采用IMAP的情况下，它永远要载入所有邮件消息的信头，POP则会传送整个邮箱。要确保您的用户理解了删除消息或者在本地文件夹中过滤它们的重要性。

> [从位于www.eudora.com/qpopper的Qualcomm可以下载POP3协议当前版本的一个不错实现。qpopper是个POP服务器，它包含了在服务器和客户机之间的TLS/SSL验证功能，并且加密邮件消息。](http://xn--www-xi9dllmz.eudora.com/qpopper%E7%9A%84Qualcomm%E5%8F%AF%E4%BB%A5%E4%B8%8B%E8%BD%BDPOP3%E5%8D%8F%E8%AE%AE%E5%BD%93%E5%89%8D%E7%89%88%E6%9C%AC%E7%9A%84%E4%B8%80%E4%B8%AA%E4%B8%8D%E9%94%99%E5%AE%9E%E7%8E%B0%E3%80%82qpopper%E6%98%AF%E4%B8%AAPOP%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%8C%E5%AE%83%E5%8C%85%E5%90%AB%E4%BA%86%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%92%8C%E5%AE%A2%E6%88%B7%E6%9C%BA%E4%B9%8B%E9%97%B4%E7%9A%84TLS/SSL%E9%AA%8C%E8%AF%81%E5%8A%9F%E8%83%BD%EF%BC%8C%E5%B9%B6%E4%B8%94%E5%8A%A0%E5%AF%86%E9%82%AE%E4%BB%B6%E6%B6%88%E6%81%AF%E3%80%82)
您可以在Internet上找到许多别的用于Linux的POP3服务器程序，务必选择其中得到积极维护的一种。
[IMAP服务器软件可从www.washington.edu/imap获得。除了把适当的IMAP项加入到/etc/services和/etc/inetd.conf文件，并确保防火墙（如果有的话）不会妨碍它工作之外，不需要进行任何配置。IMAP在过去有过安全问题的记录，请参考CERT的报告并务必得到最新的版本，如果针对您的发行版本有安全报告的话特别要注意。](http://xn--imapwww-c64ksl362a9ud81jvq2djf1h.washington.edu/imap%E8%8E%B7%E5%BE%97%E3%80%82%E9%99%A4%E4%BA%86%E6%8A%8A%E9%80%82%E5%BD%93%E7%9A%84IMAP%E9%A1%B9%E5%8A%A0%E5%85%A5%E5%88%B0/etc/services%E5%92%8C/etc/inetd.conf%E6%96%87%E4%BB%B6%EF%BC%8C%E5%B9%B6%E7%A1%AE%E4%BF%9D%E9%98%B2%E7%81%AB%E5%A2%99%EF%BC%88%E5%A6%82%E6%9E%9C%E6%9C%89%E7%9A%84%E8%AF%9D%EF%BC%89%E4%B8%8D%E4%BC%9A%E5%A6%A8%E7%A2%8D%E5%AE%83%E5%B7%A5%E4%BD%9C%E4%B9%8B%E5%A4%96%EF%BC%8C%E4%B8%8D%E9%9C%80%E8%A6%81%E8%BF%9B%E8%A1%8C%E4%BB%BB%E4%BD%95%E9%85%8D%E7%BD%AE%E3%80%82IMAP%E5%9C%A8%E8%BF%87%E5%8E%BB%E6%9C%89%E8%BF%87%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98%E7%9A%84%E8%AE%B0%E5%BD%95%EF%BC%8C%E8%AF%B7%E5%8F%82%E8%80%83CERT%E7%9A%84%E6%8A%A5%E5%91%8A%E5%B9%B6%E5%8A%A1%E5%BF%85%E5%BE%97%E5%88%B0%E6%9C%80%E6%96%B0%E7%9A%84%E7%89%88%E6%9C%AC%EF%BC%8C%E5%A6%82%E6%9E%9C%E9%92%88%E5%AF%B9%E6%82%A8%E7%9A%84%E5%8F%91%E8%A1%8C%E7%89%88%E6%9C%AC%E6%9C%89%E5%AE%89%E5%85%A8%E6%8A%A5%E5%91%8A%E7%9A%84%E8%AF%9D%E7%89%B9%E5%88%AB%E8%A6%81%E6%B3%A8%E6%84%8F%E3%80%82)
卡内基梅隆大学（Carnegie Mellon University）开发了一种叫做Cyrus IMAP的IMAP服务器，它也支持POP协议。比起UW（译者注：华盛顿大学，University of Washington）的IMAP实现来，我们更喜欢前者。
Dovecot是更新的一种软件包，它实现了IMAP和POP服务。它按照严格和明确的编码规范来编写代码，所以至少从理论上说，提高了它的安全性。Dovecot还有一些有意思的功能，例如能够把电子邮件保存在SQL数据库而不是文件系统中。Dovecot尚未取得Cyrus那样的业绩和使用基础，但它肯定是一个值得关注和评估的项目。
我们举例的所有Linux发行版本都带有叫做imapd的IMAP服务器，还有一个客户端fetchmail，它既支持IMAP协议也支持POP协议。Red Hat的imapd是CMU的Cyrus IMAP服务器，SUSE、Debian和Ubuntu使用UW的版本。Red Hat也带有pop 3d，它是POP服务器。SUSE包含3种POP服务器（这可不是多余）：qpopper（SUSE把它叫做popper），pop2d和pop3d。Debian有几种用IMAP管理邮箱的工具，命令man -kimap会告诉您它们的名字。

## 0x03 鳴謝

[18.3 邮件基本原理](http://www.5dmail.net/html/2008-4-27/200842730128.htm)
