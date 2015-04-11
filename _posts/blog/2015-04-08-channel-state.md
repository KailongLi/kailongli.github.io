---
layout:     post
title:      ONOS中Channel状态机分析
category: blog
description: 分析代码中Channel状态机，状态的变化范围从控制器开始监听，到交换机与控制器完成握手（交换机可用），使用状态机使得代码清晰易懂。
---

首先分析状态机代码的结构和语法，然后分析各个状态的变化，及其涉及到的OF消息的含义。

## 状态机代码的结构和语法分析

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

对于第二个问题：采用倒推的思路，在什么情况下可以复写非abstract方法呢？答案是一个类extends另外一个类。于是，可以得出WAIT_HELLO extends ChannelState，并且WAIT_HELLO是static final修饰的。

## 状态机各个状态变化分析
![channel state machine](/images/githubpages/channel state machine.png)

上面这张图channel状态机的流转图，下面将解释这张图的含义。

在ChannelState为INIT状态下，交换机与控制器还没有建立连接，在这个状态下，收到的任何OF消息都是不正常的。由于processOFError和processOFPortStatus这两个方法是定义的abstract，所以这两个方法必须复写。对于收到其他的OF消息，统一采用默认的处理illegalMessageReceived(h, m)。

在ChannelState为WAIT_HELLO状态下，控制器主要处理接收到的OFHello消息，进行版本协商过程。在OFChannelHandler.channelConnected方法中，控制器注释了发送OFHello消息的代码，而只是等待交换机主动发送OFHello消息，这样做的好处是简化流程，避免一些可能出现的异常场景，这也是需要交换机在与控制器建立连接后，能够主动发送OFHello消息的能力。处理OFHello消息的具体流程如下图
![WAIT_HELLO_processOFHello](/images/githubpages/WAIT_HELLO_processOFHello.png)
除了OFHello消息之外，如果收到来自交换机的Asynchronous消息(PacketIn/FlowRemoved/PortStatus)，这是有可能的，因为OF通道建立好了之后，交换机状态发生变化，那么就会发送这些Asynchronous消息给控制器，但是控制器收到之后处理不了，因为控制器还没有和交换机握手成功了；如果收到来自交换机的Symmetric消息（Error）

[netty]:http://www.importnew.com/7669.html "netty"
[状态机模式]:http://www.importnew.com/7669.html "状态机模式"
[版本协商]:http://flowgrammable.org/sdn/openflow/state-machine/ "版本协商"
