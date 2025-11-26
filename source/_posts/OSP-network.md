## OSP的网络

---
title: OSP的网络
mathjax: true
copyright: true
comment: true
date: 2025-11-26 22:15:53
categories:
- "Ops "
tags:
- "Linux "
- "OpenStack "
- "OpenShift "
- "OVN "
---

<center><img src="https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_ovngateway_on_OCP_with_memo.jpg" width=50% /></center>

### 0x01 起因

事情的起因是，客户的场景没有ovnController, 并且结合了SRIOV/DPDK的场景。但是没有见过所以用此文记录一下其中的详情。

<!-- more -->

因为场景是RHOSO，首先要明确以下概念
```
Neutron network --+-- 东西流量 private network (tenant network) 
                  |
                  +-- 南北流量 provider network --+-- non-external provider network 
                                         |
                                         +-- external provider network
```
- 东西流量是指 instance-instance 流量 
- 南北流量是指 instance-external 流量 --- 本文重点

### 0x02 北向中 OVN Controller 的工作类型

- 有ovnController --- 连接instance需要通过OVN Gateway等组件转发到二层网络
- 无ovnController --- 连接instance需要通过provider network例如BGP等二层网络

而进一步细分的话
```
neutron network --+-- private network (tenant network)
                  |
                  +-- provider network --+-- non-external provider network                 +-- ovnController -- physical network
                                         |                                  (vlan,flat) ---+
                                         +-- external provider network                     +-- physical network
                 
```

1. 有ovnController 

- OCP 节点作为OVN Gateway，则OVN controller需要运行在OCP节点
![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_ovngateway_on_OCP.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_ovngateway_on_OCP.jpg)
```
https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_networking_services/configuring-ovn-gateways_rhoso-cfgnet#ovn-gateway-ctrl-plane-dedicated-nic_configuring-gateways

ovnController:

spec:
  ...
ovn:
  template:
    ovnController:
      networkAttachment: tenant
      nicMappings:
         <network_name: nic_name>
```
- 单独的networker node作为 OVN Gateway, 则需要额外部署Network nodes 
![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_with_Networker_node.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_with_Networker_node.jpg)
```
Chapter 5. Configuring Networker nodes | Configuring networking services | Red Hat OpenStack Services on OpenShift | 18.0 | Red Hat Documentation
https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_networking_services/configure-networker-nodes_rhoso-cfgnet#proc_creating-an-OpenStackDataPlaneNodeSet-CR-networker-preprovisioned_networkers


apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
metadata:
  name: networker-nodes 
  namespace: openstack
spec:
  env: 
    - name: ANSIBLE_FORCE_COLOR
      value: "True"
```


1. 无ovnController
![https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_without_ovn_gateway.jpg](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/RHOSO_18.0_ml2_OVN_without_ovn_gateway.jpg)   
```
https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_networking_services/configuring-ovn-gateways_rhoso-cfgnet#ovn-gateway-ctrlplane-none_configuring-gateways

移除所有ovnController内容
```

### 0x03 南向中 neutron 的使用

较为复杂的北向结构探讨完了，以下是南向的一些情况
```
neutron network --+-- private network (tenant network)                      1
                  | 
                  +-- provider network --+-- non-external provider network  2
                                         |                                 
                                         +-- external provider network      3
                 
```

从上到下，创建它们的命令如下网络
```
[1]
openstack network create internal-network --internal
```
```
[2]
openstack network create public --provider-network-type vlan --provider-physical-network datacentre --provider-segment 104
```
```
[3]
openstack network create public --provider-network-type vlan --provider-physical-network datacentre --provider-segment 104 --external
```

## 0x04 结语

对OVN的整体架构有了更深刻的认知。


