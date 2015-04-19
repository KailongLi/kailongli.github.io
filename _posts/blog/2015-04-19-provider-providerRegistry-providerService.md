---
layout:     post
title:      ONOS中收到OF消息后，分发消息流程分析
category: blog
description: ONOS控制器与交换机建立连接之后，通过OF消息的交互，达到了Channel状态机的稳定状态，这时候收到OF消息之后，将分发消息到各个模块处理。
---

首先介绍代码是如何走到消息处理的方法，然后分析分发消息的规则的代码的结构和语法，最后分析这样设计代码的好处。

## 代码如何走到消息处理的方法

![dispatch message function](/images/githubpages/dispatch message function.png)

<li>① OFChannelHandler在Channel状态WAIT_DESCRIPTION_STAT_REPLY中调用Controller类的getOFSwitchInstance方法来获取交换机实例;</li>
<li>② Controller类的getOFSwitchInstance方法，则调用DriverManager的静态方法getSwitch来获取不同厂家实现的交换机实例，获取的原则是基于OFDescStatsReply消息中的“厂家信息”和“硬件信息”。注意：在实例化交换机之后，将OpenFlowAgent实例赋给了该交换机，其中OpenFlowAgent实例的产生是在OpenFlowControllerImpl类的内部类OpenFlowSwitchAgent类实例化的，见⑤。</li>
<li>③ 这里的driver意味着对接不同厂家的OF交换机，上图只是列出了少数几种OF交换机，例如：支持OF_13的OVS，OF_10的OVS和光的OF交换机</li>
<li>④ 这里的OF交换机都继承了AbstractOpenFlowSwitch的handleMessage方法，而handleMessage方法调用了就是agent的processMessage方法，然后processMessage方法调用了processPacket方法，在processPacket方法中，就是处理消息分发的分发口</li>

## 分析分发消息的规则的代码的结构和语法
### 分发消息的规则
<ul> 在 processPacket 方法中：
<li>当收到 OFPortStatus 消息，通知OpenFlowSwitchLister的portChanged方法。</li>
<li>当收到OFFeaturesPeply消息，通知OpenFlowSwitchLister的switchChanged方法。</li>
<li>当收到OFPacketIn消息，通知PacketListener的handlePacket方法。</li>
<li>当收到OFFlowRemoved消息和OFError消息，交给OpenFlowEventListener的handleMessage来处理。</li>
<li>当收到OFBarrierReply消息，交给OpenFlowEventListener的handleMessage来处理。</li>
<li>当收到OFExperimenter消息，仅仅处理experimenter为0x748771的扩展光消息，最后交给交给 OpenFlowSwitchListener 的 portChanged 来处理。</li>   

当收到OFStatsReply消息时，根据其消息类型来进行分类处理：

<li>如果类型为`PortDesc`，通知OpenFlowSwitchLister的switchChanged方法；</li>
<li>如果类型为Flow，首先构造OFFlowStateReply消息，然后调用OpenFlowEventListener的handleMessage来处理这个消息；</li>
<li>如果类型为Group，首先构造 OFGroupStatsReply 消息，然后调用OpenFlowEventListener的handleMessage来处理这个消息；</li>
<li>如果类型为GROUP_DESC，首先构造 OFGroupDescStatsReply 消息，然后调用OpenFlowEventListener的handleMessage来处理这个消息；</li>
<li>如果类型为PORT，直接调用OpenFlowEventListener的handleMessage来处理这个消息。</li>
</ul>

### 代码结构
由上面的分发规则，我们可以看出，许多OF消息都交给了不同的listener的不同方法来进行处理，例如OFSwitchListener的portChanged和switchChanged方法，OpenFlowEventListener的handleMessage方法，PacketListener的handlePacket方法等。下面主要分析各个APP如何拿到这些OF消息，并且同一个消息在各个APP之间处理的先后顺序是如何定义的。

![packet-in-from-south-to-app](/images/githubpages/packet-in-from-south-to-app.png)
以 Packet-in 处理为例来说明上面两个问题，

#### South-OF Layer 
控制器收到OFPacketIn的消息后，遍历所有的 ofPacketListener 容器中的 Packetlistener ，调用其中的 handlePacket 方法来处理 Packet-in 消息，

    for (PacketListener p : ofPacketListener.values()) {
        p.handlePacket(pktCtx);
    }
那么问题来了，ofPacketListener 中的 listener 什么时候加进来的呢？在 OpenFlowController 接口中提供了 addPacketListener 方法，以便往 ofPacketListener 容器中加入 PacketListener 。

#### Provider Layer 
OpenFlowPacketProvider 类的属性 controller 实例化 OpenFlowController，采用 OSGI 的注解方式，来添加对 OpenFlowControllerImpl 的引用

    @Reference(cardinality = ReferenceCardinality.MANDATORY_UNARY)
    protected OpenFlowController controller;
然后在 OpenFlowPacketProvider 启动的时候，将实例化的 listener 加入到 ofPacketListener 容器中。

    @Activate
    public void activate() {
        providerService = providerRegistry.register(this);
        controller.addPacketListener(20, listener);
        log.info("Started");
    }
其中 OpenFlowPacketProvider 的内部类 InternalPacketProvider 实现 Packetlistener 接口， handlePacket 方法：

    @Override
    public void handlePacket(OpenFlowPacketContext pktCtx) {
        DeviceId id = DeviceId.deviceId(Dpid.uri(pktCtx.dpid().value()));

        DefaultInboundPacket inPkt = new DefaultInboundPacket(
                new ConnectPoint(id, PortNumber.portNumber(pktCtx.inPort())),
                pktCtx.parsed(), ByteBuffer.wrap(pktCtx.unparsed()));

        DefaultOutboundPacket outPkt = null;
        if (!pktCtx.isBuffered()) {
            outPkt = new DefaultOutboundPacket(id, null,
                    ByteBuffer.wrap(pktCtx.unparsed()));
        }

        OpenFlowCorePacketContext corePktCtx =
                new OpenFlowCorePacketContext(System.currentTimeMillis(),
                        inPkt, outPkt, pktCtx.isHandled(), pktCtx);
        providerService.processPacket(corePktCtx);
    }
发现最后还是调用 providerService 的 processPacket 方法，在 OpenFlowPacketProvider 启动的时候， `providerService = providerRegistry.register(this);` 这里涉及到 `provider` 、`providerRegistry` 和 `providerService` 这三个概念，它们主要为了管理 `provider` ，下篇博客将详细讲解它的机制。就现在这种情况， providerRegistry 实例化为 PacketManager ， providerService 实例化为 PacketManager 的内部类 InternalPacketProviderService ，而这两个类属于 Core Layer，因此，调用的 InternalPacketProviderService 的 processPacket 方法。

#### Core Layer
在 InternalPacketProviderService 的 processPacket 方法中，遍历 processors 容器中所有的 PacketProcessor 的 process 方法来处理。

    @Override
    public void processPacket(PacketContext context) {
        // TODO filter packets sent to processors based on registrations
        for (PacketProcessor processor : processors.values()) {
            processor.process(context);
        }
    }
这种代码的结构（红色部分）类似于 South-OF Layer 到 Provider Layer 的跨层调用结构（紫色部分），这里将不再赘述。

#### App Layer
在 App Layer 中，提供了两个 APP 作为例子来说明第二个问题，对于同一个 OF 消息，各个 APP 处理的先后顺序，在 addProcessor 方法中，除了添加各个 APP 的 processor 之外，还有另外一个属性，那就是优先级，用来处理调用顺序的问题。在 ProxyArp 这个 APP 中，优先级为 PacketProcessor.ADVISOR_MAX + 1 

    packetService.addProcessor(processor, PacketProcessor.ADVISOR_MAX + 1);
在 BGPRouter 这个 APP 中，优先级为 PacketProcessor.ADVISOR_MAX + 4

    packetService.addProcessor(processor, PacketProcessor.ADVISOR_MAX + 4);
优先级越低，越先处理，其原理是利用了 [HashMap][] 的有序性，对于同优先级的 processer ，后加入的 processer 将覆盖前面的 processor 。




[HashMap]:http://stackoverflow.com/questions/24372257/implementing-priority-queue-using-hashmap "HashMap"

