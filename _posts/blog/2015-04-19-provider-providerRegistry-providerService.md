---
layout:     post
title:      ONOS 南向抽象层分析
category: blog
description: 南向抽象层由网络组件构成，例如交换机、主机或是链路。ONOS的南向抽象层将每个网络组件表示为通用格式的对象。通过这个抽象层，分布式核心可以维护网络组件的状态，并且不需要知道底层设备的具体细节。
---

首先引用[ONOS白皮书中篇之ONOS架构][]中描述的一段话作为开篇，南向抽象层由网络组件构成，例如交换机、主机或是链路。ONOS的南向抽象层将每个网络组件表示为通用格式的对象。通过这个抽象层，分布式核心可以维护网络组件的状态，并且不需要知道底层设备的具体细节。总之，分布式核心可以实现南向接口协议和设备无感知。这个网络组件抽象层允许添加新设备和协议，以可插拔的形式支持扩展，插件根据规格映射（或翻译）通用网络组件描述或操控设备，反之亦然。所以，南向接口确保ONOS控管多个不同的设备，即使它们使用不同的协议（OpenFlow、NetConf等）。

南向接口的分层结构如图3所示，最底层是网络设备，ONOS通过协议与设备连接，协议细节被网络组件插件或适配器屏蔽。事实上，南向接口的核心是在不知道具体协议细节和网络组件的条件下维护网络组件对象（设备、主机、链路）。通过适配层API，分布式核心可以与网络组件对象状态保持一致，适配层API将分布式核心与协议细节和网络组件相隔离。

![onos-tiers](/images/githubpages/onos-tiers.png)

<ul> 南向抽象层的主要优势包括：
<li>用不同的协议管理不同的设备，不会对分布式核心造成影响。</li>
<li>扩展性强，可以在系统中添加新的设备和协议。</li>
<li>轻松地从传统设备转移到支持OpenFlow的白牌设备。</li>
</ul>

## 代码如何实现抽象层分析
在上篇博客 [ONOS中收到OF消息后，分发消息流程分析](http://kailongli.github.io/dispatch-message/) 代码结构中，
![packet-in-from-south-to-app](/images/githubpages/packet-in-from-south-to-app.png)
South-OF Layer 和 Provider Layer 之间、Core Layer 和 App Layer 之间的解耦已经十分清楚，本节主要介绍 Core Layer 和 Provider Layer 之间的解耦，关键在于理解 `provider` 、`providerRegistry` 和 `providerService` 这三个概念。在ONOS代码中，onos\docs\src\main\javadoc\doc-files\onos-subsystem.png 这张图比较抽象地介绍了他们的关系。
![southbound abstract layer analysis](/images/githubpages/southbound abstract layer analysis.png)
首先在 `provider` 中， `provider` 向 `providerRegistry` 注册自己，来拿到 `providerService` ，从前面几篇博客分析了，南向协议层通过协议来感知网元层的变化，而 `provider` 则通过实现协议层提供的接口来实现感知协议层的消息（[上篇博客](http://kailongli.github.io/dispatch-message/)的消息分发），并且将不同协议的消息通过 `providerService` （通过注册自己拿到的服务） 来转为抽象的 Device, Host, Link, Flow, Packet 消息，这样有两个好处：一是屏蔽了不同协议层的设备；二是通过采用注册的方式，实现了代码的解耦。 

下面以Device为例，说明上面的抽象图。
![southbound abstract layer analysis-device-example](/images/githubpages/southbound abstract layer analysis-device-example.png)
我们看到 `provider` 、`providerRegistry` 和 `providerService` 这三个接口都有一个默认的实现抽象类作为基类，还有一个网络组件（ Device, Host, Link, Flow, Packet ）的扩展接口，这里是 Device 的接口， 最后有个实现类来继承基类和实现特定网络组件的接口，这种类的对称性使得代码十分的优雅。

其中注册是通过在 OpenFlowDeviceProvider 类的代码体现的：

    @Activate
    public void activate() {
        providerService = providerRegistry.register(this);
        ……
    }

其中感知则是通过 OpenFlowDeviceProvider 的内部类 InternalDeviceProvider 来体现的，我们可以看到 InternalDeviceProvider 实现南向OF协议层的两个接口 OpenFlowSwitchListener, OpenFlowEventListener 这两个接口，来监听南向的 OpenFlow 的消息，最后无缝地转化到抽象的 `providerService` 方法。以感知交换机端口为例：
南向协议层对应的是 OpenFlowSwitchListener 的  public void portChanged(Dpid dpid, OFPortStatus status); 在 InternalDeviceProvider 类中，转为抽象的 `providerService` 方法 void portStatusChanged(DeviceId deviceId, PortDescription portDescription); 区别是 portChanged 方法只针对 OpenFlow 消息，而 portStatusChanged 方法则是与协议无关的处理，既可以是感知的 OpenFlow 消息，又可以是 netconf 消息，又或者是 ovsdb 消息。

    @Override
    public void portChanged(Dpid dpid, OFPortStatus status) {
        PortDescription portDescription = buildPortDescription(status);
        providerService.portStatusChanged(deviceId(uri(dpid)), portDescription);
    }




[ONOS白皮书中篇之ONOS架构]:http://www.sdnlab.com/6800.html "ONOS白皮书中篇之ONOS架构"
[HashMap]:http://stackoverflow.com/questions/24372257/implementing-priority-queue-using-hashmap "HashMap"

