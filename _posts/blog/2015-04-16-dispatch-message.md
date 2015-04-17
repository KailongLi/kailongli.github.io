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

![packet processor](/images/githubpages/packet processor.png)

控制器启动的入口函数为

    enum ChannelState {
        ① WAIT_HELLO(false) {
            @Override
            void processOFHello(OFChannelHandler h, OFHello m)
                    throws IOException {
                // TODO We could check for the optional bitmap, but for now
                // we are just checking the version number.
                if (m.getVersion() == OFVersion.OF_13) {
                    log.debug("Received {} Hello from {}", m.getVersion(),
                            h.channel.getRemoteAddress());
                    h.sendHandshakeHelloMessage();
                    h.ofVersion = OFVersion.OF_13;
                } else if (m.getVersion() == OFVersion.OF_10) {
                    log.debug("Received {} Hello from {} - switching to OF "
                            + "version 1.0", m.getVersion(),
                            h.channel.getRemoteAddress());
                    OFHello hi =
                            h.factory10.buildHello()
                                    .setXid(h.handshakeTransactionIds--)
                                    .build();
                    h.channel.write(Collections.singletonList(hi));
                    h.ofVersion = OFVersion.OF_10;
                } else {
                    log.error("Received Hello of version {} from switch at {}. "
                            + "This controller works with OF1.0 and OF1.3 "
                            + "switches. Disconnecting switch ...",
                            m.getVersion(), h.channel.getRemoteAddress());
                    h.channel.disconnect();
                    return;
                }
                h.sendHandshakeFeaturesRequestMessage();
                h.setState(WAIT_FEATURES_REPLY);
            }
            @Override
            void processOFFeaturesReply(OFChannelHandler h, OFFeaturesReply  m)
                    throws IOException, SwitchStateException {
                illegalMessageReceived(h, m);
            }
            ……
        };
        ……

        ② private final boolean handshakeComplete;
        ChannelState(boolean handshakeComplete) {
            this.handshakeComplete = handshakeComplete;
        }

        ③ void processOFMessage(OFChannelHandler h, OFMessage m)
                throws IOException, SwitchStateException {
            switch(m.getType()) {
            case HELLO:
                processOFHello(h, (OFHello) m);
                break;
            case BARRIER_REPLY:
                processOFBarrierReply(h, (OFBarrierReply) m);
                break;
            case ECHO_REPLY:
                processOFEchoReply(h, (OFEchoReply) m);
                break;
            ……
            default:
                illegalMessageReceived(h, m);
                break;
            }
        }

        void processOFHello(OFChannelHandler h, OFHello m)
                throws IOException, SwitchStateException {
            // we only expect hello in the WAIT_HELLO state
            illegalMessageReceived(h, m);
        }

        // no default implementation. Every state needs to handle it.
        abstract void processOFPortStatus(OFChannelHandler h, OFPortStatus m)
                throws IOException, SwitchStateException;
    }
分析ChannelState这个枚举类的结构：这里首先包含用来表示状态的枚举值WAIT_HELLO。然后定义了handshakeComplete这个属性，用来标记是否完成控制器与交换机的握手过程，由于它是final类型的，所以它必须由构造函数来初始化，于是WAIT_HELLO(false)表示在WAIT_HELLO这个状态下，还没有完成握手。最后定义了processOFMessage(OFChannelHandler h, OFMessage m)这个方法，它用来根据不同的OF消息来进行不同的处理。注意处理方法的不同定义，processOFPortStatus为abstract类型，processOFHello则不是，于是，有两个疑问：
<ul>
    <li>为什么有些处理方法要声明为abstract呢？</li>
    <li>为什么在① WAIT_HELLO中可以复写processOFHello这个非abstract方法呢？</li>
</ul>
对于第一个问题：注释已经说得很明白，在各个状态下收到的OF消息中，有些OF消息有默认的处理方法，而有些OF消息则没有默认的处理方法。

对于第二个问题：采用倒推的思路，在什么情况下可以复写非abstract方法呢？答案是一个类extends另外一个类。于是，可以得出WAIT_HELLO(enum实例)像一个独特的类，并且WAIT_HELLO extends ChannelState。《JAVA编程思想》里描述：编译器不允许我们将一个enum实例当做class类型，因为每个enum元素（WAIT_HELLO）是一个ChannelState类型的static final实例，同时，由于它们是static实例，无法访问外部类的非static元素或方法，所以对于内部的enum实例而言，其行为与一般的内部类并不相同。除了实现abstract方法以外，程序员还可以覆盖常量相关的方法。

## 状态机各个状态变化分析
![channel state machine](/images/githubpages/channel state machine.png)

上面这张图channel状态机的流转图，下面将解释这张图的含义。

在ChannelState为INIT状态下，交换机与控制器还没有建立连接，在这个状态下，收到的任何OF消息都是不正常的。由于processOFError和processOFPortStatus这两个方法是定义的abstract，所以这两个方法必须复写。对于收到其他的OF消息，统一采用默认的处理illegalMessageReceived(h, m)。

在ChannelState为WAIT_HELLO状态下，控制器主要处理接收到的OFHello消息，进行版本协商过程。在OFChannelHandler.channelConnected方法中，控制器注释了发送OFHello消息的代码，而只是等待交换机主动发送OFHello消息，这样做的好处是简化流程，避免一些可能出现的异常场景，这也是需要交换机在与控制器建立连接后，能够主动发送OFHello消息的能力。处理OFHello消息的具体流程如下图
![WAIT_HELLO_processOFHello](/images/githubpages/WAIT_HELLO_processOFHello.png)
除了OFHello消息之外，如果收到来自交换机的Asynchronous消息(PacketIn/FlowRemoved/PortStatus)，这是有可能的，因为OF通道建立好了之后，交换机状态发生变化，那么就会发送这些Asynchronous消息给控制器，但是控制器收到之后处理不了，因为控制器还没有和交换机握手成功了；如果收到来自交换机的Symmetric消息OFError，直接断开连接；如果收到来自交换机的Symmetric消息OFEchoRequest，则相应地回OFEchoReply；如果收到来自交换机的Symmetric消息OFEchoReply，控制器什么都不处理；如果收到Controller-to-Switch信息（OFFeaturesReply和OFStatisticsReply），则断开连接。

在ChannelState为WAIT_FEATURES状态下，控制器主要处理接收到的OFFeaturesReply消息，一是获取DPID，二是将OFFeaturesReply消息暂存起来，以供后期判断交换机类型，获取相应的交换机实例。处理OFFeaturesReply消息的具体流程如下图
![WAIT_FEATURES_REPLY_processOFFeaturesReply](/images/githubpages/WAIT_FEATURES_REPLY_processOFFeaturesReply.png)
对于OF10协议，先要下发Controller-to-Switch类型的OFSetConfig消息（消息中的miss_send_len为OFPCML_NO_BUFFER,即0xffff），其目的是让交换机能够上报完整的packet_in报文，注意：这里上报到控制器的packet_in报文是指没有用output action到OFPP_CONTROLLER的场景；然后下发Controller-to-Switch类型的OFBarrierRequest消息，保证OFSetConfig消息先下发完成；最后下发OFGetConfigRequest消息，由于前面使用OFBarrierRequest消息，这保证了OFSetConfig消息一定在OFGetConfigRequest消息前下发完成，因此在这三条消息下发完成之后，进入WAIT_CONFIG_REPLY状态，等待接收OFGetConfigReply消息，当接收到了OFGetConfigReply消息，查看OFSetConfig消息中设置的字段miss_send_len值OFPCML_NO_BUFFER,即0xffff是否成功。

* 收获：控制器给交换机下发的配置消息，如何检验是否下发成功？对于交换机能够回复的消息，那么等待交换机回复的消息来确认控制器给交换机下发的消息是否成功；对于交换机不回复的，则采用OFBarrierRequest来保证控制器下发的消息完成，然后下发查询消息，等待交换机回复消息，再来确认控制器下发的配置消息是否成功。

对于OF13协议，下发OFPortDescRequest消息，然后将控制器的状态设置为WAIT_PORT_DESC_REPLY，在WAIT_PORT_DESC_REPLY中处理流程如下：
![WAIT_PORT_DESC_REPLY_processOFStatisticsReply](/images/githubpages/WAIT_PORT_DESC_REPLY_processOFStatisticsReply.png)
最后回到了WAIT_CONFIG_REPLY，对比OF_13比OF_10多了一个WAIT_PORT_DESC_REPLY的状态，是因为控制器实例化交换机之后，要将其端口描述信息赋给交换机。然而在OF_10的OFStatsReply中却没有端口描述信息这种消息。

在ChannelState为WAIT_CONFIG_REPLY状态下，控制器收到OFGetConfigReply之后，检查其中的miss_send_len是否为0xffff，然后下发OFStatsRequest消息，进入WAIT_DESCRIPTION_STAT_REPLY状态。

在ChannelState为WAIT_DESCRIPTION_STAT_REPLY状态下，控制器收到OFStatsReply之后，
![WAIT_DESCRIPTION_STAT_REPLY_processOFStatisticsReply](/images/githubpages/WAIT_DESCRIPTION_STAT_REPLY_processOFStatisticsReply.png)
判断其类型是否为description，如果不是，则继续等待下一个OFStatsReply消息，如果是的话，根据OFStatsReply消息的mfr_desc(Manufacturer description)和hw_desc（Hardware description）判断该交换机是Open vSwitch还是其他硬件交换机，以初始化交换机实例，并且将之前暂存的OFFeaturesReply和OFPortDescReply消息赋给交换机，另外还包括一些属性：之前协商好的版本号，控制器与交换机建立连接的通道Channel，已经建立连接标志这三个属性都赋给交换机。实例化交换机之后，接着进行交换机特定的handshake过程，比如支持OF13的OVS需要发送OFBarrierRequest消息，支持OF10的OVS需要发送OFFlowAdd消息，之后再判断该交换机实例是否已经存在，如果已经存在，则断开连接，如果不存在，则处理之前状态缓存的所有OFPortStatus消息，将状态转为ACTIVE。

在ChannelState为ACTIVE状态下，控制器收到各种OF消息之后，将OF消息进行分发，交给其他模块进行处理。下一篇博客将分析这种分发机制。







[netty]:http://www.importnew.com/7669.html "netty"
[状态机模式]:http://www.importnew.com/7669.html "状态机模式"
[版本协商]:http://flowgrammable.org/sdn/openflow/state-machine/ "版本协商"
