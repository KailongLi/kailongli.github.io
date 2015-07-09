---
layout:     post
title:      ONOS OpenFlow控制器主备角色分析
category: blog
description: 当交换机连接控制器集群后，交换机同时连接多个控制器，控制器集群会从这几个控制器中进行选举，以便为交换机选举一个Master角色的控制器。
---

当交换机连接控制器集群后，交换机同时连接多个控制器实例，控制器集群会从这几个实例中为该交换机选择一个 Master 角色的控制器。整个过程主要分为三步：

* 本地控制器请求集群为该控制器实例对交换机的角色分配
* 下发 OpenFlow Role Request 消息，建立本地控制器与交换机的关系
* 发布 DeviceEvent 

下面按照这三步展开
** 角色分配
角色分配主要分为两步：第一步是形成候选人列表 candidateMap ，第二步是从候选人中选取 leader 。 

    Versioned<List<NodeId>> candidates = candidateMap.computeIf(path,
        currentList -> currentList == null || !currentList.contains(localNodeId),
        (topic, currentList) -> {
            if (currentList == null) {
                return ImmutableList.of(localNodeId);
            } else {
                List<NodeId> newList = Lists.newLinkedList();
                newList.addAll(currentList);
                newList.add(localNodeId);
                return newList;
            }
        });
这是形成候选人列表的计算表达式，其中 path(由deviceId形成) 作为 key ，候选人列表作为 value 。