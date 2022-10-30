---
title: 每天5分钟OSP-Glance02
mathjax: true
copyright: true
comment: true
date: 2022-10-29 23:14:37
updated: 2022-10-29 23:14:37
categories:
- "Ops "
tags:
- "Linux "
- "Storage "
- "OpenStack "
- "Glance "
- "每天五分鐘玩轉OpenStack "
---

<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221029000932.png" width=50% /></center>

## 0x00️⃣前言

本文继续记录一些个人的glance学习笔记。

<!-- more -->

## 0x01️⃣webui操作image

下载地址： http://download.cirros-cloud.net/

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221029144641.png)

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221029150152.png)

**关于Public ，Private，Shared和Community的区别如下。**

- **Public images**

  These are images provided by the OpenStack administrators. They are visible to all users.

  镜像由OpenStack管理员提供。所有用户可见。

- **Private images**

  A private image is owned by a specific project and cannot be viewed or used by other projects.

  私有镜像由特定项目所有。不可被其他项目查看或使用。

- **Shared images**

  A shared image is a private image that can be viewed/used by specific other projects that the image owner adds as “members” to the image.

  共享镜像是一个私有镜像，可由镜像所有者作为“成员”添加到镜像的特定其他项目查看/使用。

- **Community images**

  A community image is an image uploaded by a project, and such project wants other projects use such image, but isn’t interested in maintaining a relationship with these tenants by making them image members.

  社区镜像是项目上传的镜像，该项目希望其他项目使用该镜像，但避免了繁杂的租户的关系。

**点击我们上传的cirros超链接，就跳转下面页面。**

![](https://img.madebug.net/m4d3bug/images-of-website/master/blog/20221029153615.png)

## 0x02️⃣CLI操作image

cirros比较小，webui交互体感较好，但如果较大的image则CLI可以方便后台运行，上传个RHEL9看看

~~~shell
(overcloud) [stack@rhosp16-director ~]$ openstack image create rhel90 --file /tmp/rhel-baseos-9.0-x86_64-dvd.iso --disk-format iso --container-format bare
(overcloud) [stack@rhosp16-director ~]$ openstack image show rhel90 --max 80
+------------------+-----------------------------------------------------------+
| Field            | Value                                                     |
+------------------+-----------------------------------------------------------+
| checksum         | b1156c2533d191a4a752e99195d03471                          |
| container_format | bare                                                      |
| created_at       | 2022-10-29T08:46:10Z                                      |
| disk_format      | iso                                                       |
| file             | /v2/images/7374f85c-c34e-4061-aa4a-5262ee2c4df3/file      |
| id               | 7374f85c-c34e-4061-aa4a-5262ee2c4df3                      |
| min_disk         | 0                                                         |
| min_ram          | 0                                                         |
| name             | rhel90                                                    |
| owner            | 5ea1e912713542b58cfbfa56c3a93645                          |
| properties       | direct_url='swift+config://ref1/glance/7374f85c-c34e-4061 |
|                  | -aa4a-5262ee2c4df3', os_hash_algo='sha512', os_hash_value |
|                  | ='ff546a297276df19ae31ebc0adbd901989d4f4c4397d0abedd7a7a6 |
|                  | afa772dac45cec1f06479919426ca5a24e93e8b588a32a88f032739c7 |
|                  | 9a5c2c5709df6a08', os_hidden='False',                     |
|                  | stores='default_backend'                                  |
| protected        | False                                                     |
| schema           | /v2/schemas/image                                         |
| size             | 8579448832                                                |
| status           | active                                                    |
| tags             |                                                           |
| updated_at       | 2022-10-29T08:55:58Z                                      |
| virtual_size     | None                                                      |
| visibility       | shared                                                    |
+------------------+-----------------------------------------------------------+
~~~

默认情况下，红帽OpenStack 16使用swift来负责glance的存储。

~~~shell
(overcloud) [stack@rhosp16-director ~]$ openstack user list
+----------------------------------+-------------------------+
| ID                               | Name                    |
+----------------------------------+-------------------------+
| 06733500c8244a448305aa92c4f4cdae | admin                   |
| b709c40050d94ca49651cbd1e83f48d3 | cinder                  |
| 3fd1abb25a1d47f39391f4fee2b83184 | cinderv2                |
| a73a4566d2654a4cbd8f1b20aba915c9 | cinderv3                |
| 5fad2deebe5842fd8fc01d8ef32e4d3d | glance                  |
| 7746848c40f247d689c44d95acb11d76 | heat                    |
| 1a5499776a294737999d1002f04c0652 | heat_stack_domain_admin |
| 703b975c2c1f4c78a44e24e769dd27d0 | heat-cfn                |
| 5bee2b21ecea4bcda81b38f361269626 | neutron                 |
| d0c42ea85066412795e789dcb6fc7468 | nova                    |
| a4a8959ecf8b454cad9d39d2964514bf | octavia                 |
| 6511c586fb5a4f39b6ffb0caa47b3db1 | placement               |
| 73895f5ffad74abaab18108db8933b4c | swift                   |
+----------------------------------+-------------------------+
~~~

## 0x03️⃣红帽OpenStack命令剖析

**命令范式**

~~~
CMD <obj>-create [parm1] [parm2]…
CMD <obj>-delete [parm]
CMD <obj>-update [parm1] [parm2]…
CMD <obj>-list
CMD <obj>-show [parm]
~~~

- Glance

  - glance版本

    ~~~
    glance image-create
    glance image-delete
    glance image-update
    glance image-list
    glance image-show
    ~~~

  - 红帽openstack版本，直接拿掉了obj，flavor同理

    ~~~
    openstack image create
    openstack image delete
    openstack image update
    openstack image list
    openstack image show
    ~~~

**被操作对象都有ID**

**help查看用法**

## 0x04️⃣如何Troubleshooting

glance日志主要是glance_api.log 和glance_registry.log。由于红帽OpenStack采用容器的方式来进行分发，所以路径可以通过podman inspect来得知是`/var/log/containers/glance`

## 0x05️⃣总结感悟

记录一下每天五分钟OpenStack的读书笔记，顺便试试新格式。意外发现一个非官方的[社区OpenStack文档]()，还不错。

