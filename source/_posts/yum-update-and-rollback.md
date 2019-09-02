---
title: Yum Update and Rollback
mathjax: true
copyright: true
comment: true
date: 2019-08-29 22:32:25
updated: 2019-08-29 22:32:25
categories:
- "Ops "
tags:
- "RHEL 7 "
- "YUM "
photo:
---

*In this post, I am going to markdown that YUM provides commands for updating to the specified version, and how to rollback the action (not suggest to downgrade the system version If not necessarily).*

## *Environmental preparation*

### *Trun off the selinux*

``` nohighlight
[root@localhost ~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
```

## *Yum update*

### *Specify the version you need*

```nohighlight
[root@localhost ~]# yum update --releasever=7.6
```

*Everything just done.*

## *Rollback yum update*

### *Listed the history in your machine*

```nohighlight
[root@localhost ~]# yum history list all
Loaded plugins: langpacks, product-id, search-disabled-repos, subscription-
              : manager
ID     | Login user               | Date and time    | Action(s)      | Altered
-------------------------------------------------------------------------------
     8 | root <root>              | 2019-08-29 11:17 | E, I, O, U     |  504 EE
     7 | root <root>              | 2019-08-28 14:13 | I, U           |  181 EE
     6 | root <root>              | 2019-08-23 12:48 | Install        |    1   
     5 | root <root>              | 2019-08-23 12:48 | Erase          |    1   
     4 | root <root>              | 2019-08-23 10:46 | Update         |    1   
     3 | root <root>              | 2019-08-23 10:41 | Install        |    1   
     2 | root <root>              | 2019-07-25 11:12 | Update         |    2   
     1 | System <unset>           | 2019-07-25 10:45 | Install        | 1363   
```

### *Specify the action to rollback*

```nohighlight
[root@localhost ~]# yum history undo 8
```

## *Handling instability error*

*This mechanism is intented to protect users from ending up with a system without systemd.*

```nohighlight
...
--> Finished Dependency Resolution
Error: Trying to remove "systemd", which is protected
[root@localhost ~]# mv /etc/yum/protected.d/systemd.conf /tmp/
```

## *Done*

*In an environment where the system needs to updated, and the host's service is not too much, it must be clear that the rollback will inevitably affect the following:*

+ *dbus*
+ *kernel*
+ *glibc (dependencies of glibc such as gcc)*
+ *selinux-policy* *

*Although you still need to use caution when you have a snapshot.*
