---
title: sendmail的深入瞭解II
mathjax: true
copyright: true
comment: true
date: 2020-10-02 14:21:27
categories:
- "Ops "
tags:
- "sendmail "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/LinuxAdministrationHandbook.jpg" width=50% /></center>

本文簡單記錄Email自身以及其參與的工作机制閱讀筆記。

<!-- more -->
## 0x00 組成部分

- 信封（envelope）

    > 信封确定消息将投递到哪里，或者如果消息不能投递出去的话，应该把它返回给谁。对于单个收件人来说，信封地址通常与信头的From和To行一致，如果消息是发送给一个邮递列表的，那么就不一致了。地址是单独提供给MSA的。信封对用户不可见，它不是消息本身的一部分，它供sendmail在内部确定把这则消息发送到哪里。

- 信頭（header）

    > 信头是一组格式遵循RFC822的属性/值。它们记录与消息有关的各种信息，比如发送的日期和时间、它在旅程中经过哪些传输代理传递等。信头是邮件消息的一个真实部分，只不过用户代理向用户显示消息时通常会隐藏一些不太引人注意的项。

- 消息主體（body of message）

    > 消息的主体是将要发送的实际内容。它必须由纯文本组成，虽然文本经常表示各种二进制内容，但此时采用了对邮件来说是安全的编码方式。

## 0x01 郵件尋址

- 本地尋址標識：用戶登錄名
- Internet尋址標識：user@host.domain user@domain
- 過時的地址類型：(中繼轉發)

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/OldEmailAddressType.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/OldEmailAddressType.png)

## 0x02 郵件信頭

- 標準信頭的格式在[RFC822](https://tools.ietf.org/html/rfc822)中定义，也可自定義“X-”信頭(被郵件系統忽略)來隨消息傳送。
- 用戶代理(MUA)，傳輸代理(MTA)各負責部分信頭。
- 負責的部分如下：

    ![https://img.madebug.net/m4d3bug/images-of-website/master/blog/header.png](https://img.madebug.net/m4d3bug/images-of-website/master/blog/header.png)

- DA於最後一次投遞中添加最上面的From。
- Return-Path: 回傳報錯信息。
- Delivery-Date: 發送日期
- Received: 經過的MTA會依次添加。127.0.0.1表明它是來自local的MTA。包含其他字段如下：
    - 發送機器名稱
    - 接收機器名稱
    - 接收機器的MTA版本(8.13.1)
    - 接收機器的id唯一標識號(k612inkG001576)
    - 收件人
    - 日期時間
    - 本地時區與UTC偏移量
- Message-ID: 消息ID，世界範圍內郵件系統是唯一的。UA追加
- 每次經過MTA都需要dns查詢來完成Received。

![https://img.madebug.net/m4d3bug/images-of-website/master/blog/MTADNSQuery.jpg](https://img.madebug.net/m4d3bug/images-of-website/master/blog/MTADNSQuery.jpg)

## 0x03 結語

讀書筆記，Email在傳輸中參與眾多工序，有點像流水線作業。這在大型的郵件系統中尤為重要。

## 0x04 鳴謝

[18.2 剖析邮件消息](http://www.5dmail.net/html/2008-4-27/200842725951.htm)
