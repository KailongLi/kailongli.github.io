---
layout:     post
title:      ONOS OpenFlow控制器主备角色分析
category: blog
description: 当交换机连接控制器集群后，控制器会进行选举，以便为交换机选举一个Master角色的控制器。
---

本篇博客首先介绍DeviceEvent的处理流程的例子，然后分析各个模块的含义，最后抽象事件处理流程。

## 事件处理流程
![event processing](/images/githubpages/event processing.png)

上面的图有些小，请见谅。事件处理的流程如下：

①DeviceManager启动时调用EventDeliveryService的addSink方法将处理DeviceEvent的EventSink (AbstractListenerRegistry)添加进来。

②CoreEventDispatcher启动时一直不停地从Event队列中，取事件，然后根据事件的类的类型，来获取相应的EventSink，然后调用eventSink 的process方法来处理事件，这是消费者部分。

③BgpRouter的内部类InnerDeviceListener实现DeviceListener，主要为了实现处理event的event方法。

④BgpRouter调用DeviceService接口的addListener方法来将InnerDeviceListener实例添加进来。

⑤DeviceManager实现DeviceService接口的addListener方法，调用AbstractListenerRegistry类的addListener方法，将InnerDeviceListener放入一个listeners集合中。

⑥DeviceManager类的内部类InternalDeviceProviderService，感知到Device消息后，调用EventDispatcher接口的post(Event)方法来投递事件。注意在EventDispatcher接口的实现类CoreEventDispatcher中实现post方法，只是将event添加到Event队列中，至此，生产-消费者模式的生产部分已经完成。

⑦结合②，eventSink的process方法遍历所有的listeners集合，调用listener的event方法，也就是BgpRouter的内部类实现的event方法。

## 各个模块的含义

#### EventListener<E extends Event> 
EventListener 接口中，只提供了一个 event 方法，它用来对特定事件作出特定的处理。其中特定的事件是由泛型 <E extends Event> 来的， DeviceListener 接口的定义如下：

    public interface DeviceListener extends EventListener<DeviceEvent> {
    }

特定的处理则是由各个APP来实现 DeviceListener 接口的 event 方法，注意这里有个小技巧，各个APP，例如 BgpRouter ，使用内部类 InnerDeviceListener 来实现 DeviceListener ，为什么要使用内部类来实现 DeviceListener 呢？如果只是需要一个接口的引用，为什么不通过外围类来实现那个接口呢？这个问题从《JAVA编程思想》中得到答案：

> 每个内部类都能够独立地继承自一个（接口的）实现，所以无论外围类是否已经继承了某个（接口的）实现，对于内部类都没有影响。

如果没有内部类提供的、可以继承多个具体的或抽象的类的能力，一些设计与编程问题就很难解决。从这个角度看，内部类使得多重继承的解决方案变得完整。接口解决了部分问题，而内部类有效地实现了“多重继承”。也就是说，内部类允许继承多个非接口类型。举个例子：在 TopologyMetrics 这个类中，假如它需要同时监听 Device 接口和 Host 接口，那么它需要实现 DeviceListener 和 HostListener 这两个接口，而这两个接口都拥有同样的方法 event ，如果仅用外部类来实现这两个接口，那么一个 event 方法会覆盖另外一个 event 方法，因此只能用内部类，来独立地实现某个接口。

总结 EventListener<E extends Event> 的作用：
* 它给其他特定事件接口（例如DeviceEventListener）来继承出特定的 EventListener；
* 继承出来的接口给不同APP来实现，override event 方法，来实现对 event 的特定处理。

#### EventSink 
EventSink主要提供处理特定事件类型处理方法的能力，在ONOS代码中只有 AbstractListenerRegistry 实现 EventSink 接口。注意 AbstractListenerRegistry 是一个泛型类，

    public class AbstractListenerRegistry<E extends Event, L extends EventListener<E>> implements EventSink<E> {
        ……
    }

在 AbstractListenerRegistry 类中，实现事件处理的 process 方法实际上是调用了 EventListener 的 event 方法，具体的做法是遍历 EventListener 集合中所有的 listener ，EventListener集合即 EventListener Sink，因此 AbstractListenerRegistry 类还需要提供将其他 listener 添加到 EventListener Sink 中的方法 addListener , 注意对于特定类型的事件， EventListener 集合为特定类型的 EventListener 集合。

总结 EventSink （AbstractListenerRegistry）的作用：
* 为不同的事件提供 EventSink 的模板；
* EventSink 提供往 EventListener Sink 中添加 EventListener 的方法；
* 封装 EventListener Sink 中所有 EventListener 处理 event 的方法。

#### EventDeliveryService 
EventDeliveryService 、EventDispatcher 、 EventSinkRegistry 这三个接口的关系如下：

    public interface EventDeliveryService extends EventDispatcher, EventSinkRegistry {
    }

EventDeliveryService 继承了 EventDispatcher 和 EventSinkRegistry 的作用，并且对外提供接口，提供投递事件的接口和 EventSink 的注册机制。EventSinkRegistry 接口的默认实现  DefaultEventSinkRegistry 说明了 EventSink 的注册机制：

    @Override
    public <E extends Event> void addSink(Class<E> eventClass, EventSink<E> sink) {
        checkNotNull(eventClass, "Event class cannot be null");
        checkNotNull(sink, "Event sink cannot be null");
        checkArgument(!sinks.containsKey(eventClass),
                      "Event sink already registered for %s", eventClass.getName());
        sinks.put(eventClass, sink);
    }

根据事件类型来注册 EventSink ，一种事件类型只能有一个 EventSink ，那么解注册方法也是这样的机制，另外还提供两个方法：根据事件类型获取 EventSink 和 获取所有事件类型的方法。

由最上的类图可知： 事件处理机制的核心 CoreEventDispatcher 实现了接口 EventDeliveryService 和继承了 EventSinkRegistry 接口的默认实现  DefaultEventSinkRegistry ，它采用的是生产-消费者模式， post 方法用来将事件添加到事件队列，作为生产者一部分，另外内部类 DispatchLoop 类，不停地从事件队列中取出事件来进行处理，作为消费者一部分。

总结 EventDeliveryService（ CoreEventDispatcher ）的作用：
* 提供EventSink注册机制；
* 作为事件处理机制的核心，实现生产-消费者模式。

经过以上分析，总结出事件对象模型关系：
![event model](/images/githubpages/event model.png)

## 抽象事件处理流程
![Event processing procedure](/images/githubpages/Event processing procedure.png)