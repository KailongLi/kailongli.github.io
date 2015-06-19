---
layout:     post
title:      ONOS 强一致性算法raft的java实现半copycat代码分析之leader election
category: blog
description: ONOS中强一致性算法raft的java版实现采用的是copycat，它包括leader election、 Log replication 、 Cluster membership changes 、 Log compaction 这四个特性，本文主要介绍其中最常用也是最简单的 leader election。
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

ONOS中**一致性**主要分为两种类型：一种是**强一致性**，其要求当一个实例更新网络状态时任何实例随后的读操作都返回最近更新的数值；另一种是**最终一致性**，它属于弱一致性，当系统保证如果没有新的状态更新时，最终所有的实例都能获得最后的更新保持最终状态一致，中间允许读取操作延后一段时间。**强一致性** 算法采用的是由zookeeper-->hazelcast-->[raft][]转变的过程，而**最终一致性**算法采用的是[gossip][]  

ONOS中一致性算法的java版实现采用的是[copycat][]，它包括 Leader election、 Log replication 、 Cluster membership changes 、 Log compaction 这四个特性。本文主要分析 Leader election 特性，要分析 copycat 中这个特性的具体代码实现，有两个前提：理解 [raft][] 算法 和 掌握 java 8 新特性的部分语法，因此，文章结构主要分为三个部分：一是介绍 raft 算法；二是介绍 java 8 新特性的语法；三是具体分析 copycat 实现 Leader election 的代码。

## raft 算法
Leader election算法参考论文 ["In Search of an Understandable Consensus Algorithm"](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&ved=0CDAQFjACahUKEwi7sKWTkI_GAhUIXIgKHZtHAjM&url=https%3A%2F%2Framcloud.stanford.edu%2Fraft.pdf&ei=jmt9VbuxB4i4oQSbj4mYAw&usg=AFQjCNE8XQb0VEwFmg-Xo5yUdZpYq7BEOg&sig2=t3_OF96mCgVSiURI-MswGQ&bvm=bv.95515949,d.cGU&cad=rja)  
其中文翻译版 [《寻找一种易于理解的一致性算法》](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)   
简易描述版 [《Raft一致性算法》](http://blog.csdn.net/cszhouwei/article/details/38374603)  
raft动画，有助于理解 [《Raft动画》](http://thesecretlivesofdata.com/raft/)  
raft官方网站，含有与raft相关的详细信息 [《Raft官网》](http://thesecretlivesofdata.com/raft/) 

相信 raft 理解起来应该不难，因为 raft 算法的动机就是寻找一种更加易于理解的 Paxos 算法，关于利用随机性来实现 Leader Election 算法，除了容易理解的好处之外，其中论文中的一句话“尽管在大多数情况下我们都试图去消除不确定性，但是也有一些情况下不确定性可以提升可理解性。尤其是，随机化方法增加了不确定性，但是他们有利于减少状态空间数量，通过处理所有可能选择时使用相似的方法。”，我认为通过随机化来减少状态空间数量是一个很大的好处。在通信协议中，CSMA/CD也是采用随机回退的机制来解决信道中通信的碰撞。

## 介绍java 8 新特性
学习java 8 新特性，主要推荐一本书[Java SE 8 for the Really Impatient][]，主要学习：
Chapter 1 Lambda Expressions  
Chapter 2 The Stream API  
Chapter 3 Programming with Lambdas  
Chapter 6 Concurrency Enhancements  

这本书的练习题答案参考：[Solutions-for-exercises-from-Java-SE-8-for-the-Really-Impatient][]。  
关于java 8 这些新（黑）特（科）性（技）的加入，变化较大，同时带来的争议也较大，例如：在接口中增加默认方法和静态方法，它的主要一个动机是扩展原来的API，因此需要在之前版本的接口中增加方法，但是按照java的规则，其所有实现类必须要实现该方法，这样太麻烦，于是想出了默认方法（在接口中直接实现该方法），该接口的实现类就不用再实现该方法了，这明显改变了java接口的规则。

## copycat 中 Leader election 的实现
在开始分析 copycat 中 Leader election 实现代码之前，建议阅读[copycat][]的描述，有助于理解 Resource, Log, Cluster 等名词含义。
copycat 代码中提供了一个关于 Leader election 的例子 (LeaderElectingVerticle 类的 start 方法)，根据这个例子，梳理 Leader election 的具体流程。

#### 前期准备条件

    String address = container.config().getString("address");
    JsonArray members = container.config().getArray("cluster");

    // Configure the Copycat cluster with the Vert.x event bus protocol and event bus members. With the event
    // bus protocol configured, Copycat will perform state machine replication over the event bus using the event
    // bus addresses provided by the protocol URI - e.g. eventbus://foo
    // Because Copycat is a CP framework, we have to explicitly list all of the nodes in the cluster.
    ClusterConfig cluster = new ClusterConfig()
      .withProtocol(new VertxEventBusProtocol(vertx))
      .withLocalMember(String.format("eventbus://%s", address))
      .withMembers(((List<String>) members.toList()).stream()
        .map(member -> String.format("eventbus://%s", member)).collect(Collectors.toList()));

    // Create a leader election configuration with a Vert.x event loop executor.
    LeaderElectionConfig config = new LeaderElectionConfig()
      .withExecutor(new VertxEventLoopExecutor(vertx));  
首先从配置文件中获取localMember IP地址和 Cluster 中所有 members 的IP地址集合；然后根据这两个信息和底层传输协议 VertxEventBusProtocol 来形成 ClusterConfig 信息，注意底层传输协议是可以配置的，copycat目前实现了三种协议： Netty, Vert.x 和 Vert.x 3 (todo: 比较这三者的差异、优劣)，代码中 "->" 表示lambda表达式；最后配置 LeaderElection 的 Executor 。

#### 选举算法的实现

    // Create and open the leader election using the constructed cluster configuration.
    LeaderElection.create("election", cluster, config).open().whenComplete((election, error) -> {
      // Since we configured the election with a Vert.x event loop executor, CompletableFuture callbacks are executed
      // on the Vert.x event loop, so we don't have to use runOnContext.
      if (error != null) {
        startResult.setFailure(error);
      } else {
        this.election = election;

        // Once the election has been opened, register a message handler on the election cluster to allow other members
        // of the cluster to send messages to us if we're the leader. Messages sent to this handler will be sent over
        // the Vert.x event bus since we're using the event bus protocol.
        election.cluster().member().<String, Void>registerHandler("print", message -> {
          System.out.println(message);
          return CompletableFuture.completedFuture(null);
        });

        // Finally, register an election listener. When a member is elected leader, log a message indicating who was
        // elected leader and send the leader a message to print.
        election.addListener(member -> {
          container.logger().info(member.uri() + " was elected leader!");
          member.send("print", "Hello world!").thenRun(() -> {
            container.logger().info("Leader printed 'Hello world!'");
          });
        });

        startResult.setResult(null);
      }
    });
首先调用 LeaderElection 接口的静态方法create（黑科技），`LeaderElection.create("election", cluster, config)` ，我们来看下这个静态方法的具体代码：

    static LeaderElection create(String name, ClusterConfig cluster, LeaderElectionConfig config) {
        ClusterCoordinator coordinator = new DefaultClusterCoordinator(new CoordinatorConfig().withName(name).withClusterConfig(cluster));
        return coordinator.<LeaderElection>getResource(name, config.resolve(cluster))
          .addStartupTask(() -> coordinator.open().thenApply(v -> null))
          .addShutdownTask(coordinator::close);
    }
它是整个LeaderElection的核心代码，根据前期准备的 ClusterConfig 来实例化 ClusterCoordinator ，ClusterCoordinator 负责 Cluster 中各个 members 间的消息通信调度的核心，




[raft]:http://thesecretlivesofdata.com/raft/ "raft动画"
[gossip]:http://blog.csdn.net/aesop_wubo/article/details/20289837 "最终一致性之gossip协议"
[copycat]:https://github.com/kuujo/copycat "copycat"
[Java SE 8 for the Really Impatient]:http://pdf.th7.cn/down/files/1411/Java%20SE%208%20for%20the%20Really%20Impatient.pdf  "java 8 新特性"
[Solutions-for-exercises-from-Java-SE-8-for-the-Really-Impatient]:https://github.com/galperin/Solutions-for-exercises-from-Java-SE-8-for-the-Really-Impatient-by-Horstmann.git "Solutions-for-exercises-from-Java-SE-8-for-the-Really-Impatient"