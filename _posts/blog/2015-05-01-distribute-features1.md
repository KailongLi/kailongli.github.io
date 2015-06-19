---
layout:     post
title:      ONOS 分布式学习（一）
category: blog
description: 当交换机连接控制器集群后，控制器会进行选举，以便为交换机选举一个Master角色的控制器。
---

首先引用并且概况来自SDNLAB的文章[ONOS高可用性和可扩展性实现初探](http://www.sdnlab.com/9021.html)作为学习的入点，

## 概况文章要点

SDN定义的三个特性

* **控制平面和数据平面的分离**
在南向接口层，采用协议插件以实现控制平面与数据平面的分离。
* **逻辑上集中控制**
在东西向的扩展上，通过分布式集群的方式以实现逻辑上集中控制。
* **开放的编程接口**
在北向接口层，提供一套应用编程接口以实现网络的可编程性的应用接口。

**一致性**主要分为两种类型：一种是**强一致性**，其要求当一个实例更新网络状态时任何实例随后的读操作都返回最近更新的数值；另一种是**最终一致性**，当系统保证如果没有新的状态更新时，最终所有的实例都能获得最后的更新保持最终状态一致，中间允许读取操作延后一段时间。**强一致性** 算法采用的是由zookeeper-->hazelcast-->[raft][]转变的过程，而**最终一致性**算法采用的是[gossip][]  
下面表格是ONOS中应用状态和属性的关系，本文中以 Flow Rules 作为**最终一致性**的例子， Switch-Controller mapping 作为**强一致性**的例子来进行学习。

State|Properties 
:---------------|:---------------
Network Topology|Eventually consistent, low latency access
Flow Rules, Flow Stats|Eventually consistent, shardable, soft state
Switch-Controller mapping, Distributed Locks|Strongly consistent, slow changing
Application Intents, Resource Allocations|Strong consistent, durable, Immutable




[raft]:http://thesecretlivesofdata.com/raft/ "raft动画"
[gossip]:http://blog.csdn.net/aesop_wubo/article/details/20289837 "最终一致性之gossip协议"
