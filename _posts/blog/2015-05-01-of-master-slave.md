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

这是形成候选人列表的计算表达式，其中 ， path(由deviceId形成) 表示 candidateMap 中的 key ， currentList 表示该 key 对应的 value ， 如果 currentList 满足下面条件 `currentList == null || !currentList.contains(localNodeId)` 则根据最后一个表达式重新计算 value 并返回。整个代码段表示特定 path 所对应的候选人列表中需要包含 localNodeId 。  
修改 candidateMap 会形成一个 MapEvent ， 在 DistributedLeadershipManager 启动的时候，对 candidateMap 添加 listener 用来处理 MapEvent ， 将 MapEvent 转为 CANDIDATES_CHANGED 的 LeadershipEvent ， 交由函数 onLeadershipEvent 来处理， 在该函数中，先更新 candidateBoard （供查询当前的 candidates 列表），然后将 LeadershipEvent 抛给各个实现了 LeadershipEventListener 的 APP 处理

选举 leader 是在 DistributedLeadershipManager 类的 electLeader 函数中：  

    NodeId topCandidate = candidates
                        .stream()
                        .filter(n -> clusterService.getState(n) == ACTIVE)
                        .findFirst()
                        .orElse(null);
candidates 是一个 LinkList ，使用 stream 的好处是：直到最后一步才计算值，这样做提高效率。整个表达式的含义是找到集群中 status=ACTIVE 的第一个成员 （交换机第一个连接的控制器实例），让其作为 topCandidate ，如果 topCandidate 与 localNodeId 相等，那么角色就是 Master 否则为 Standby 。

总的来说，选取 leader 是根据 交换机连接控制器的顺序决定的，越先建立连接的控制器，优先级越高。而不是采用负载均衡的算法，不过做起来也比较简单。

** 下发 OF 消息
如果是 OF10 协议，则调用 RoleManager 的 sendNxRoleRequest 方法， 这里只有两种角色 Master 和 Other；如果是 OF13 协议，则调用  RoleManager 的 sendOF13RoleRequest 方法。注意，发送 OF_Role_Request 消息后，将 xid 和 role 放入 60 秒的缓存中，目的是检验 OF_Role_Reply 消息的正确性。

    pendingReplies.put(sendOF13RoleRequest(role), role);

** 发布 DeviceEvent 
采用 gossip 协议将该 device 同步到其他节点，并且根据是否为 create 还是 update ，返回 DEVICE_ADDED 还是 DEVICE_UPDATED 事件，供上层 APP 进行处理。





