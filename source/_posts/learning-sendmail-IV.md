---
title: sendmail的深入瞭解IV
mathjax: true
copyright: true
comment: true
date: 2020-10-04 16:32:58
categories:
- "Ops "
tags:
- "sendmail"
---
<center><img src="https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/LinuxAdministrationHandbook.jpg" width=50% /></center>

本文簡單記錄郵件別名。

<!-- more -->

## 0x00 別名類型

> 别名（alias）能够让系统管理员或单个用户重新发送邮件 。他们可以定义邮递列表、在机器之间转发邮件，或者允许用多个名字指定一个用户。别名处理是递归的，所以一个别名指向的其他目的地也可以是别名。

sendmail支持好几种别名机制：

- 純文件映射。
- 老式文件發佈系統：Sun的NIS和NIS+，Apple的NetInfo
- 各種郵件路由數據庫：LDAP（Lightweight Directory Access Protocol，轻量级目录访问协议）。

> 传统上可以在下列3个地方定义别名（遗憾的是，要用3种不同的句法）：
在用户代理的配置文件中（由发送方用户定义）；
在系统范围内起作用的/etc/mail/aliases文件中（由系统管理员定义）；
在用户的转发文件~/.forward中（由接收方用户定义） 。

以下是別名相關例子：

1. 遞歸語法，發往nemeth的應發往evi，發往evi的應發往evi@mailhub。

    ```
    nemeth:  evi
    evi: evi@mailhub
    authors: evi,garth,trent
    ```

2. 全局aliases文件路徑是/etc/aliases，/etc/mail/aliases实际上才是“标准”位置。

    local-name匹配传入消息的源地址，而收件人名单则包含收件人地址或其他别名的名字。

    ```
    local-name:  recipient1,recipient2,...
    ```

3.  /etc/mail/aliases文件**应该永远包含一个叫做“postmaster”的别名**，它把邮件转发给负责维护邮件系统的人。
4.  當Receiced行數多達25行跳(hop)轉則會回彈到發件人。
5.  別名可以是以下內容。

    > 一个包含地址列表的文件；
    一个应该把消息附加到其后的文件；
    一条应该把消息作为其输入的命令。

6.  DontBlameSendmail設置可以關閉對aliaes文件的敏感程度。

## 0x01 別名與郵遞列表

- include: 引入郵遞列表文件，方便細分權限。

    ```
    sabook:  :include:/usr/local/mail/lah.readers
    ```

- lah.readers 應位於非NFS安裝的文件系統中，格式如下：

    ```
    owner-sabook:  evi
    ```

## 0x02 發郵件給文件

- 文件名須是絕對路徑，特殊字符雙引號。文件名不認為是標準地址。

    ```
    complaints:  /dev/null
    ```

- 也可以在aliases文件或者用戶的.forward文件中使用⬆️。
- 被寫文件須對任何人可寫，有setuid位但不可執行，或由sendmail默認用戶所有。
- .forward文件中引用該文件，須對原消息收件人所有，且帶有✍️寫權限，且用戶合法，有/etc/passwd記錄，合法shell。對於由root用戶擁有的文件，以4644或4600來設置setuid。不可執行。

## 0x03 發郵件給程序

- 可以使用ftp接收標準輸入。如下：

    ```
    autoftp:  "|/usr/local/bin/ftpserver"
    ```

- ⚠️注意事项⚠️

    > 用这项功能甚至比发邮件给文件更容易造成安全性漏洞，所以再强调一次，它只允许用在aliases、.forward或:include:文件中，sendmail的默认配置现在要求使用受限制的shell—smrsh 。在aliases文件中，程序作为sendmail的默认用户运行；否则，程序作为.forward或:include:文件的所有者运行。这个用户必须已经在/etc/passwd文件中列出，并具有一个有效的shell（在/etc/shells中）。

    > 程序的邮寄程序在运行接收邮件的命令之前先把它的工作目录变为用户的主目录（或者，如果那个目录不能访问，就变为root的目录）。默认原本是sendmail的邮件队列目录，但是有些基于csh的shell不接受。

## 0x04 别名举例

![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/AliaesExample.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/AliaesExample.jpg)

postmaster指定維護郵件系統的郵件，發送到trouble，evi兩個別名。其中trouble是由trouble.alias提供郵件列表。

tmr，trouble mail readers（问题邮件阅读器），用於分配給管理員新手查看問題。

sa-class，維護包含學生列表的數據文件，類似於上述sabook一樣的子列表。

diary記錄重要的日誌事件，升級變更崩潰。

## 0x05 郵件轉發

- .forward是系統層面的變量，使用ForwardIPath變量即可開啟。
- .forward格式如下：

    ```
    evi@ipn.caida.org
    evi@atrust.com

    \mcbryan, "/home/mcbryan/archive", mcbryan@f1supi1.gmd.de
    ```

- 第一種格式，發往evi轉發給caida的機器ipn和evi@atrust.com
- 第二種格式，本地緩衝區，傳入郵件的永久存檔文件，臨時郵件地址。反斜線指代無視其他任何關於mcbryan別名設置。
- 注意⚠️設置不當引起的死循環，直至25跳後原路返回。

    ```
    全局aliases
    evi:  evi@boulder

    boulder的.forward
    evi@anchor.cs
    ```

## 0x06 別名數據庫

自帶散列數據庫，/etc/mail/aliases.db，修改aliases文件後使用newaliases自動生成。

## 0x07 鳴謝

[18.4 邮件别名](http://www.5dmail.net/html/2008-4-27/200842730420.htm)
