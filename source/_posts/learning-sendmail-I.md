---
title: sendmail的深入瞭解I
mathjax: true
copyright: true
comment: true
date: 2020-10-01 21:21:06
categories:
- "Ops "
tags:
- "Linux "
- "sendmail "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/%20LinuxAdministrationHandbook.jpg" width=50% /></center>

本文簡單記錄sendmail的工作模式模型閱讀筆記。

<!-- more -->
## 0x00 什麼是郵件系統

有以下五部分：

- 让用户阅读和撰写邮件的“邮件用户代理”（mail user agent，MUA）
- 在机器之间发送消息的“邮件传输代理”（mail transport agent，MTA）
- 把消息放到本地消息库 中的“投递代理（delivery agent，DA）”；它有时叫做本地投递代理（Local Delivery Agent，LDA）
- 可有可无的“访问代理（access agent）”，它可以把用户代理连接到消息库（例如，通过IMAP或POP协议）
- 邮件提交代理（mail submission agent，MSA），这种代理使用SMTP（simple mail transport protocol，简单邮件传输协议），并完成传输代理的一些工作。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/emailsystem.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/emailsystem.jpg)

## 0x01 用戶代理(MUA/UA)

- 用户代理的一项工作是确保在邮件消息的内容中嵌入的任何可能被邮件系统误解的文字得到保护。用作消息之间记录分隔符的字符串“From”就是一个例子。
- 一个用户代理不必和邮件系统的其余部分运行在相同的系统，乃至相同的平台上。
- /bin/mail是最初的用户代理

> Red Hat和Fedora上的/bin/mail是原UNIX命令mail的BSD版本；在SUSE、Debian和Ubuntu上，它变成了/usr/bin/mail 。这个用户代理只支持文本，并且需要在本地保存邮件。
Mozilla的Thunderbird，有Linux、Windows和Mac OS的版本。
Evolution（也叫做Novell Evolution，以前叫Ximian Evolution），有Linux、Windows和Mac OS的版本。
华盛顿大学（University of Washington）的pine，[www.washington.edu/pine。](http://www.washington.edu/pine%E3%80%82)
Qualcomm的Eudora，用于Mac机或者运行Windows的PC。
微软的Outlook，也用于Windows。

## 0x02 傳輸代理(MTA)

- 必须接受从MUA那里来的邮件，读懂目的地，并设法把邮件交给正确的主机进行投递。
- 大多数传输代理还担当消息提交代理(MSA)，完成把新消息发送到邮件系统的功能。
- 传输代理使用RFC821中定义的SMTP协议（Simple Mail Transport Protocol，简单邮件传输协议），或者使用RFC1869、RFC1870、RFC1891和RFC1985中定义的ESMTP协议（Extended SMTP，扩展的SMTP协议）。

> Red Hat、Fedora和SUSE Linux自带装好的sendmail。Debian表面上好像是带了sendmail，但是如果您仔细一看，会发现sendmail实际上是到Exim这个邮件传输代理的链接。Exim经过仔细地调整开发，已经能够理解sendmail的命令行标志。直接调用“sendmail”的用户代理应该是不知情的。Ubuntu默认带Exim。

## 0x03 郵件提交代理(MSA)

> 在繁忙的主邮件枢纽上的传输代理要花费大量时间进行邮件消息的预处理：确保所有主机名是完整的，修改从已损坏的邮件用户代理（MUA）那里得到的信头、日志记录错误、重写信头等。RFC2476引入的思路是：将邮件提交代理（MSA）从邮件传输代理（MTA）中分离出来，以便分担工作负荷并获得最佳性能。

- MSA负责消息由传输代理发送之前必须完成的所有预备工作和错误检验。它有点像在MUA和MTA之间插入一个头脑清楚的检验员。
- 如果用户代理不能识别端口587，您仍然可以在端口25上运行MSA，但是得在和MTA不同的另一台服务器。(直接打开SMTP连接的用户代理必须进行修改，使用587端口来调用MSA。)
- MSA，587端口，用-bs或者-bm标志调用。
- MTA，25端口，用-bd标志调用。

## 0x04 投遞代理(DA)

- 從MTA接受郵件，並把它真正投递给适当的本地收件人。可以是人、一个邮递列表、文件或一个程序。

> 每种类型的收件人可能需要一个不同的代理。/bin/mail是用于本地用户的投递代理。/bin/sh是发给文件或程序的邮件最初的投递代理，投递到文件则在内部处理。sendmail的新近版本附带了更安全的本地投递代理，名为mail.local和smrsh（念做“smursh”）。www.procmail.org的procmail也可以用作本地投递代理，参见18.9.16节。类似地，如果您运行Cyrus imapd作为AA（访问代理），它还包括自己的本地投递代理。

## 0x05 消息庫

- 本地计算机上保存电子邮件的地方。/var/spool/mail或/var/mail。
- 不同OS權限如下：

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/authmail.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/authmail.jpg)

- SUSE的权限要宽松一点儿，但是在邮件缓存目录中的文件模式为660，属组为root。设置了粘附位的目录（在权限位中的t）不允许用户彼此删除对方的文件，即便他们有这个目录的写权限也不行。不过，恶意的用户可以填满邮件缓存目录，把它当作一个乱写乱画的分区，或者创建另一个用户的邮箱。

## 0x06 訪問代理(AA)

- 協助用戶下載，通過IMAP（Internet Message Access Protocol，Internet消息访问协议）或者POP（Post Office Protocol，邮局协议）

## 0x07 結語

重新整理了一下閱讀筆記，未完待續。

## 0x08 鳴謝

- [18.1 邮件系统](http://www.5dmail.net/html/2008-4-27/200842725823.htm)
