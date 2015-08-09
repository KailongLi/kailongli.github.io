---
layout:     post
title:      copycat描述
category: blog
description: ONOS中强一致性算法采用的是raft，它包括leader选举、 Log replication 、 Cluster membership changes 、 Log compaction 这四个特性，copycat是raft这四个特性的java版实现。
---

Copycat
=======

[![Build Status](https://travis-ci.org/kuujo/copycat.png)](https://travis-ci.org/kuujo/copycat)

Copycat is an extensible distributed coordination framework for Java 8 built on the
[Raft consensus protocol](https://raftconsensus.github.io/).

**Over the past couple months, Copycat has undergone significant development in preparation for an initial release.
That release is tentatively planned for June 21st!**

The first release of Copycat will feature:
* High-level path-based resource API
* A well-tested standalone Raft implementation
* Support for distributed events, messaging, and remote task execution
* Essential distributed collections, e.g. map, set, list
* Essential distributed atomic variables, e.g. atomic boolean, long, and reference
* Essential distributed coordination utilities, e.g. locks and leader elections
* Support for time and session based features, including expiring or ephemeral keys
* Support for arbitrary custom resources

The first release of Copycat will also include a number of essential improvements:
* Friendly builder API for `Cluster` and `Raft`
* Support for client-only nodes
* Linearizable Raft clients via sessions
* Significant improvements to internal log with efficient compaction algorithm
* Low-level buffer abstraction for off-heap memory and disk I/O
* An efficient custom serialization framework

The current development branch is `0.6.0`. We expect these changes to be merged into the `master` branch and pushed to
Maven central in the coming days and weeks.

**Note that due to the impending changes, much of the existing documentation is obsolete. Check back soon for new documentation!**

## Deprecated

Copycat is a CP (consistent/partition-tolerant) oriented exercise in distributed coordination built on a consistent
replicated log. The core of Copycat is an extensible asynchronous framework that uses a mixture of
[gossip](http://en.wikipedia.org/wiki/Gossip_protocol) and the [Raft consensus protocol](https://raftconsensus.github.io/)
to provide a set of high level APIs that solve a variety of distributed systems problems including:
* [Leader election](#leader-elections)
* [Replicated state machines](#state-machines)
* [Strongly consistent state logs](#state-logs)
* [Eventually consistent event logs](#event-logs)
* [Distributed collections](#collections)
* [Failure detection](#failure-detection)
* [Remote execution](#remote-execution)

*Copycat requires Java 8*

User Manual
===========

1. [Introduction](#introduction)
1. [Getting started](#getting-started)
   * [Thread safety](#thread-safety)
   * [Setting up the cluster](#setting-up-the-cluster)
   * [Configuring the protocol](#configuring-the-protocol)
   * [Creating a Copycat instance](#creating-a-copycat-instance)
   * [Accessing the Copycat cluster](#accessing-the-copycat-cluster)
1. [Configuration](#configuration)
   * [Configuration files](#configuration-files)
   * [Configuration objects](#configuration-objects)
1. [Resources](#resources)
   * [Resource lifecycle](#resource-lifecycle)
   * [Configuring resources](#configuring-resources)
      * [Resource replicas](#resource-replicas)
      * [Log configuration](#log-configuration)
      * [Serialization](#serialization)
      * [Executors](#executors)
      * [Resource configuration files](#resource-configuration-files)
   * [Creating resources](#creating-resources)
   * [Resource clusters](#resource-clusters)
1. [State machines](#state-machines)
   * [Creating a state machine](#creating-a-state-machine)
   * [Creating a standalone state machine](#creating-a-state-machine-as-a-standalone-service)
   * [Configuring the state machine](#configuring-the-state-machine)
   * [Designing state machine states](#designing-state-machine-states)
   * [State machine commands](#state-machine-commands)
   * [State machine queries](#state-machine-queries)
   * [The state context](#the-state-context)
   * [Transitioning the state machine state](#transitioning-the-state-machine-state)
   * [Synchronous proxies](#synchronous-proxies)
   * [Asynchronous proxies](#asynchronous-proxies)
1. [Event logs](#event-logs)
   * [Creating an event log](#creating-an-event-log)
   * [Event log configuration](#event-log-configuration)
   * [Writing events to the event log](#writing-events-to-the-event-log)
   * [Consuming events from the event log](#consuming-events-from-the-event-log)
1. [State logs](#state-logs)
   * [Creating a state log](#creating-a-state-log)
   * [State log configuration](#state-log-configuration)
   * [State commands](#state-commands)
   * [State queries](#state-queries)
   * [Submitting operations to the state log](#submitting-operations-to-the-state-log)
   * [Snapshotting](#snapshotting)
1. [Leader elections](#leader-elections)
   * [Creating a leader election](#creating-a-leader-election)
1. [Collections](#collections)
   * [AsyncMap](#asyncmap)
   * [AsyncList](#asynclist)
   * [AsyncSet](#asyncset)
   * [AsyncMultiMap](#asyncmultimap)
   * [AsyncLock](#asynclock)
1. [The Copycat cluster](#the-copycat-cluster)
   * [Members](#members)
      * [Active members](#active-members)
      * [Passive members](#passive-members)
      * [Member states](#member-states)
   * [Leader election](#leader-election)
   * [Direct messaging](#direct-messaging)
   * [Broadcast messaging](#broadcast-messaging)
   * [Remote execution](#remote-execution)
1. [Protocols](#protocols)
   * [The local protocol](#the-local-protocol)
   * [Netty protocol](#netty-protocol)
   * [Vert.x protocol](#vertx-protocol)
   * [Vert.x 3 protocol](#vertx-3-protocol)
1. [Architecture](#architecture)
   * [Logs](#logs)
   * [Strong consistency and Copycat's Raft consensus protocol](#strong-consistency-and-copycats-raft-consensus-protocol)
      * [Leader election](#leader-election-1)
      * [Write replication](#command-replication)
      * [Read consistency](#query-consistency)
      * [Log compaction](#log-compaction)
   * [Eventual consistency and Copycat's gossip protocol](#eventual-consistency-and-copycats-gossip-protocol)
      * [Passive membership](#passive-membership)
      * [Log replication](#log-replication)
      * [Failure detection](#failure-detection)

## Introduction

Copycat is a distributed coordination framework built on the
[Raft consensus protocol](https://raftconsensus.github.io/). Copycat facilitates numerous types of strongly consistent
distributed data structures - referred to as [resources](#resources) - based on a replicated log. Each Copycat
cluster can support multiple resources, and each resource within a cluster runs a separate instance of the consensus
algorithm to perform leader election and log replication.

The Copycat [cluster](#the-copycat-cluster) consists of a core set of [active members](#active-members) which
participate in the [Raft leader election and replication protocol](#strong-consistency-and-the-raft-consensus-algorithm)
and perform all synchronous replication. Additionally, the cluster may contain a set of additional
[passive members](#passive-members) which can be added and removed from the cluster dynamically. While active members
participate in replication of logs via Raft, passive members
[perform replication asynchronously using a simple gossip protocol](#eventual-consistency-and-copycats-gossip-protocol).

The following image demonstrates the relationship between active and passive members in the Copycat cluster:

![Copycat cluster](http://s23.postimg.org/i8xyzg2ez/Copycat_Cluster_New_Page_2.png)

Active members participate in synchronous log replication via the Raft consensus protocol and ultimately gossip
committed log entries to passive members, while passive members gossip among each other.

For more information on Copycat's leader election and replication implementation see the in depth explanation of
[Copycat's architecture](#architecture).

In addition to supporting multiple resources within the same cluster, Copycat supports different cluster configurations
on a per-resource basis. This allows Copycat's resources to be optimized by partitioning resources and assigning
different partitions to different members in the cluster.

The following image depicts the partitioning of resources across the Copycat cluster:

Each [resource](#resources) in the cluster has its own related logical `Cluster` through which it communicates with
other members of the [resource's cluster](#resource-clusters). Just as each resource performs replication for its
associated [log](#logs), so too does each resource cluster perform leader elections independently of other resources
in the cluster. Additionally, Copycat's global and resource clusters can be used to send arbitrary messages between
members of the cluster or execute tasks on members remotely.

## Getting started

The Copycat project is structured as several Maven submodules.

The primary module for Copycat is the `copycat-api` module. The Copycat API module aggregates all of the resource
modules into a single API, and allows multiple distributed resources to be created within a single Copycat instance.

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-api</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

Additionally, the following resource modules are provided:
* `copycat-event-log` - A replicated log designed for recording event histories
* `copycat-state-log` - A replicated log designed for strongly consistent recording of state machine
  command histories
* `copycat-state-machine` - A replicated state machine framework built on top of `copycat-state-log` using proxies
* `copycat-leader-election` - A simple leader election framework built on top of `copycat-state-log`
* `copycat-collections` - A set of strongly consistent distributed collection APIs built on top of `copycat-state-log`

Each of the Copycat resource modules can be used independently of one another and independently of the high level
`copycat-api`.

#### Thread safety

All of Copycat's user-facing APIs are thread safe. Additionally, each of Copycat's [resources](#resources) and their
related [clusters](#resource-clusters) are executed on a single thread-per-resource.

### Configuring the protocol

Copycat supports a pluggable protocol that allows developers to integrate a variety of frameworks into the Copycat
cluster. Each Copycat cluster must specific a `Protocol` in the cluster's `ClusterConfig`. For more information
see the section on [protocols](#protocols).

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol());
```

Alternatively, users can override the default protocol by overriding `cluster-defaults.conf` with `cluster.conf`.

`cluster.conf`

```
protocol {
  class: net.kuujo.copycat.netty.NettyTcpProtocol
  send.buffer.size: 8192
  receive.buffer.size: 8192
}
```

Copycat core protocol implementations include [Netty](#netty-protocol), [Vert.x](#vertx), and [Vert.x 3](vertx-3).
Each protocol implementation is separated into an independent module which can be added as a Maven dependency.

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-netty</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

The `copycat-core` module also contains a `LocalProtocol` which can be used for multi-threaded testing. When using
the `LocalProtocol` for testing, you should use the same `LocalProtocol` instance across all test instances.

### Setting up the cluster

In order to connect your `Copycat` instance to [the Copycat cluster](#the-copycat-cluster), you must add a set of
protocol-specific URIs to the `ClusterConfig`. The cluster configuration specifies how to find the *active* members -
the core voting members of the Copycat cluster - by defining a simple list of active members URIs. Passive members are
not defined in the cluster configuration. Rather, they are added dynamically to the cluster via a gossip protocol.
For more about active members see the section on [cluster members](#active-members).

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new VertxTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:1234")
  .withMembers("tcp://123.456.789.0:1234", "tcp://123.456.789.1:1234", "tcp://123.456.789.2:1234");
```

Note that the cluster's `local-member` attribute *must be defined*, but it does not have to be represented in the list
of configured active members. If the local member is not in the active members list, the node will join the cluster as
a *passive* member.

Alternatively, users can configure the cluster by overriding the default cluster configuration with a `cluster.conf`
[Typesafe configuration](https://github.com/typesafehub/config) file:

`cluster.conf`

```
protocol {
  class: net.kuujo.copycat.vertx.VertxTcpProtocol
  connect.timeout: 30000
}

members: [
  "tcp://123.456.789.0:1234"
  "tcp://123.456.789.1:1234"
  "tcp://123.456.789.2:1234"
]
```

Note that member URIs must be unique and all members of the cluster must agree with the configured protocol. Member URIs
will be checked at configuration time for validity.

Because of the design of the [Raft algorithm](#strong-consistency-and-copycats-raft-consensus-protocol), it is strongly
recommended that any Copycat cluster have *at least three active voting members*. Raft requires a majority of the
cluster to be available in order to accept writes. Since passive members don't participate in the Raft protocol, only
active members contribute towards this membership count. So, for instance, a cluster of three active members can
tolerate one failure, a cluster of five active members can tolerate two failures, and so on.

### Creating a Copycat instance

To create a `Copycat` instance, call one of the overloaded `Copycat.create()` methods:
* `Copycat.create()` - creates a Copycat instance with the file-based cluster configuration
* `Copycat.create(ClusterConfig cluster)` - creates a Copycat instance with a custom cluster configuration
* `Copycat.create(CopycatConfig config)` - create a Copycat instance with a custom configuration

Note that the first argument to any `Copycat.create()` method is a `uri`. This is the protocol specific URI of the
*local* member, and it may or may not be a member defined in the provided `ClusterConfig`. This is because Copycat
actually supports eventually consistent replication for clusters much larger than the core Raft cluster - the cluster
of *active* members defined in the cluster configuration.

```java
Copycat copycat = Copycat.create(cluster);
```

When a `Copycat` instance is constructed, a central replicated state machine is created for the entire Copycat cluster.
This state machine is responsible for maintaining the state of all [cluster resources](#resources). In other words, it
acts as a central registry for other log based structures created within the cluster. This allows Copycat to
coordinate the creation, usage, and deletion of multiple log-based resources within the same cluster.

Once the `Copycat` instance has been created, you must open the `Copycat` instance by calling the `open` method.

```java
CompletableFuture<Copycat> future = copycat.open();
future.get();
```

When the `Copycat` instance is opened, all the [resources](#resources) defined in the
[configuration](#configuring-resources) will be opened and begin participating in leader election and log replication.

### Accessing the Copycat cluster
Each `Copycat` instance contains a reference to the global `Cluster`. The cluster handles communication between
Copycat instances and provides a vehicle through which users can [communicate](#messaging) with other Copycat instances
as well. To get a copy of the Copycat `Cluster` simply call the `cluster` getter on the `Copycat` instance.

```java
Cluster cluster = copycat.cluster();
```

For more information on using the `Cluster` object to communicate with other members of the cluster see the section
on [the Copycat cluster](#the-copycat-cluster).

## Configuration

All Copycat configuration types - `CopycatConfig`, `ClusterConfig`, `ResourceConfig`, etc - are backed by the excellent
[Typesafe Config](https://github.com/typesafehub/config) library.

Configuration for all configurable objects - cluster, logs, resources, and serializers - are backed by default
configuration files. Configurations use a hierarchical resolution process that allows users to override portions of
configurations by defining specifically named configuration files. In the documentation that follows, each section
regarding a configurable type will indicate how configurations can be overridden.

In general, all configurations are resolved with the following precedence:
* Code based configuration (e.g. `eventLogConfig.withRetentionPolicy(new SizeBasedRetentionPolicy(1024 * 1024));`)
* User defined file-based configuration (e.g. `my-event-log.conf`)
* Type specific file configuration (e.g. `event-log.conf`, `resource.conf`)
* Default type specific file configuration (e.g. `event-log-defaults.conf`, `resource-defaults.conf`)

Additionally, Copycat's configuration resolution process supports namespaced configurations for all configurable types.
So, users can define configuration namespaces by separating configuration sections in the resource name with a `.`. For
example, `new StateMachineConfig("foo.bar.baz")` will search for and resolve configuration files in the following order:
* `foo.bar.baz.conf`
* `foo.bar.conf`
* `foo.conf`
* `state-machine.conf`
* `state-machine-defaults.conf`
* `state-log.conf`
* `state-log-defaults.conf`
* `resource.conf`
* `resource-defaults.conf`

Additionally, configurable resources referenced within the given configuration will be resolved in a similar order.
For instance, the state machine's underlying `Log` will be resolved in the following order:
* The `log` section of `foo.bar.baz.conf`
* The `log` section of `foo.bar.conf`
* The `log` section of `foo.conf`
* The `log` section of `state-machine.conf`
* The `log` section of `state-machine-defaults.conf`
* The `log` section of `state-log.conf`
* The `log` section of `state-log-defaults.conf`
* `log.conf`
* `log-defaults.conf`

To learn more about Copycat's configuration format, users are encouraged to read the
[Typesafe Config documentation](https://github.com/typesafehub/config) and particularly the
[HOCON configuration format](https://github.com/typesafehub/config/blob/master/HOCON.md)

## Resources

Each Copycat instance can support any number of various named resources. *Resource* is an abstract term for all of the
high level log-based data types provided by Copycat. Ultimately, each resource - whether is be an
[event log](#event-logs), [state log](#state-logs), [state machine](#state-machines), or [collection](#collections) -
is backed by a Raft replicated [log](#logs) that is managed by the Copycat cluster coordinator internally.

### Resource lifecycle

When a Copycat instance is created and opened, any resources for which the local `Copycat` instance is listed as a
replica will be opened internally and immediately begin participating in leader election and replication for that
resource. Leader election and replication for resources occur separately from the global Copycat cluster. Therefore,
the leader for a given resource may frequently differ from the global Copycat cluster leader. This ensures that Copycat
can maintain consistency across multiple resources within the same cluster.

### Configuring resources

Each of Copycat's resource types - `EventLog`, `StateLog`, `StateMachine`, etc - has an associated `Config` class.
This configuration class is used to configure various attributes of the resource, including Raft-specific configuration
options such as election timeouts and heartbeat intervals as well as resource specific configuration options. Each of
these configuration classes can be used to configure their respective resources when the resources are created

```java
EventLogConfig config = new EventLogConfig()
  .withSerializer(KryoSerializer.class)
  .withLog(new FileLog()
    .withSegmentSize(1024 * 1024));

EventLog<String> eventLog = copycat.createEventLog("event-log", config);
```

Each resource can optionally define a set of replicas separate from the global cluster configuration. The set of
replicas in the resource configuration *must be contained within the set of active members defined in the cluster
configuration*. The resource's replica set defines the members that should participate in leader election and
synchronous replication for the resource. Therefore, in a five node Copycat cluster, if a resource is configured with
only three replicas, writes will only need to be persisted on two nodes in order to be successful.

```java
CopycatConfig config = new CopycatConfig()
  .withClusterConfig(new ClusterConfig()
    .withProtocol(new NettyTcpProtocol())
    .withLocalMember("tcp://123.456.789.1:1234")
    .withMembers(
      "tcp://123.456.789.0:1234",
      "tcp://123.456.789.1:1234",
      "tcp://123.456.789.2:1234",
      "tcp://123.456.789.3:1234",
      "tcp://123.456.789.4:1234"
    ));

Copycat copycat = Copycat.create(config).open().get();

EventLogConfig eventLogConfig = new EventLogConfig()
  .withSerializer(KryoSerializer.class)
  .withReplicas("tcp://123.456.789.1:1234", "tcp://123.456.789.2:1234", "tcp://123.456.789.3:1234")
  .withLog(new FileLog()
    .withSegmentSize(1024 * 1024));

EventLog<String> eventLog = copycat.createEventLog("event-log", eventLogConfig);
```

If replicas are defined for the resource and the local cluster member is not specified in the list of resource members,
the local resource will join the clustered resource as a *passive* member of the resource.

#### Resource replicas

Each of the resources in Copycat's cluster do not have to be replicated on all [active members](#active-members) on
the core cluster. Instead, Copycat allows replicas to be configured on a per-resource basis. This allows users to
configure replication for a specific resource on a subset of the core cluster of active members.

```java
CopycatConfig config = new CopycatConfig()
  .withClusterConfig(new ClusterConfig()
    .withMembers(
      "tcp://123.456.789.0:5000",
      "tcp://123.456.789.1:5000",
      "tcp://123.456.789.2:5000",
      "tcp://123.456.789.3:5000",
      "tcp://123.456.789.4:5000"));

EventLogConfig eventLogConfig = new EventLogConfig()
  .withReplicas(
    "tcp://123.456.789.1:5000",
    "tcp://123.456.789.2:5000",
    "tcp://123.456.789.3:5000"));
```

#### Log configuration

Each replica of each resource in the Copycat cluster writes to a separate, configurable log. Logs are defined in the
resource configuration via the `setLog` and `withLog` methods. Copycat provides several different log implementations
for different use cases:
* `FileLog` - A simple `FileChannel` based log. This is the default log for most Copycat data structures.
* `BufferedLog` - An in-memory log implementation built on `TreeMap`.
* `ChronicleLog` - A very efficient [Chronicle Queue](https://github.com/OpenHFT/Chronicle-Queue) based log.

Many resource configuration options involve configuring the log itself. Users can configure the performance and
reliability of all Copycat logs by tuning the underlying file log configuration options, for instance, by configuring
the frequency with which Copycat flushes logs to disk.

```java
EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog()
    .withFlushInterval(60, TimeUnit.SECONDS));
```

Additionally, configurations for all logs or for specific logs can be overridden via
[Typesafe configuration](https://github.com/typesafehub/config) files. To override the default configuration for all
logs, define a `log.conf` configuration resource on the classpath. Alternatively, to configure a named log define
a configuration resource (i.e. `*.conf`, `*.json`, or `*.properties`) on the classpath and reference that resource
when constructing the log. For example:

`my-log.conf`

```
segment.interval: 60000
flush.on-write: true
```

Given a custom configuration resource on the classpath, simply pass that resource name to the log constructor:

```java
EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog("my-log"));
```

Log configurations are resolved with the following order of precedence:
* Code based configuration
* Custom file based configuration
* `log.conf`, `log.json`, or `log.properties`
* `log-defaults.conf`

Additionally, log configurations can be overridden within resource configuration files as well.

```
log {
  class: net.kuujo.copycat.log.FileLog
  segment.interval: 60000
  flush.on-write: true
}
```

#### Serialization

By default, entries to Copycat's logs are serialized using the default [Kryo](https://github.com/EsotericSoftware/kryo)
based serializer, so users can implement custom serializer for log entries by implementing `KryoSerializable`:

```java
public class MyEntry implements KryoSerializable {

  public void write (Kryo kryo, Output output) {
    // ...
  }

  public void read (Kryo kryo, Input input) {
     // ...
  }
}
```

To register custom classes for serialization by the Kryo serializer, create a `serializer.conf` on the classpath and
define classes and their IDs in the `registrations` object:

`serializer.conf`

```
registrations.1 = com.mycompany.myproject.MySerializableClass
registrations.2 = com.mycompant.myproject.OtherSerializableClass
```

In order to ensure portability between members, it's important that all nodes in the Copycat cluster contain the same
registrations with the same identifiers.

Serializer configurations can also be specified on a per-resource basis:

`my-resource.conf`

```
serializer {
  class: net.kuujo.copycat.util.serializer.KryoSerializer
  registrations {
    1: com.mycompany.myproject.MySerializableClass
    2: com.mycompany.myproject.OtherSerializableClass
  }
}
```

#### Executors

Copycat makes heavy use of Java 8's `CompletableFuture`. Each Copycat resource guarantees that completable future
callbacks will be executed on the same thread-per-resource. However, in some cases the user needs control over the
context in which resource callbacks are executed. For instance, when running Copycat within Vert.x, it can be useful
to execute `CompletableFuture` callbacks, election result handlers, message handlers, and other callbacks on the
Vert.x event loop. Copycat allows users to configure the `Executor` used by each resource instance.

To configure the `Executor` instance for a resource, add the executor to the resource's configuration:

```java
EventLogConfig config = new EventLogConfig()
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

All user facing callbacks within the resource and its related [cluster](#resource-clusters) will be executed with
the given `Executor` instance.

To configure the executor for the global Copycat cluster and instance, add an `Executor` to the `CopycatConfig`:

```java
CopycatConfig config = new CopycatConfig()
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

Note that because of the semantics of threads, executors are the one resource that *can not be specified via
configuration files*.

#### Resource configuration files

All resource configurations inherit from the base `ResourceConfig` configuration class. The `ResourceConfig` receives
defaults from the `resource-defaults.conf` [Typesafe configuration](https://github.com/typesafehub/config) file. To
override default resource configuration options for all resource types, define a `resource.conf` on the classpath. Any
configuration options specified in `resource.conf` will override equivalent values defined in `resource-defaults.conf`.

`resource.conf`

```
# Set the default resource log to BufferedLog
log.class: net.kuujo.copycat.log.BufferedLog
```

### Creating resources

With resources configured and the `Copycat` instance created, resources can be easily retrieved by calling any
of the resource-specific methods on the `Copycat` instance:
* `<T> EventLog<T> createEventLog(String name)`
* `<T> StateLog<T> createStateLog(String name)`
* `<T> StateMachine<T> createStateMachine(String name)`
* `LeaderElection createLeaderElection(String name)`
* `<K, V> AsyncMap<K, V> createMap(String name)`
* `<K, V> AsyncMultiMap<K, V> createMultiMap(String name)`
* `<T> AsyncList<T> createList(String name)`
* `<T> AsyncSet<T> createSet(String name)`
* `AsyncLock createLock(String name)`
* `AsyncLong createLong(String name)`
* `AsyncBoolean createBoolean(String name)`
* `AsyncReference createReference(String name)`

```java
Copycat copycat = Copycat.create(config);

copycat.open().thenRun(() -> {
  StateMachine<String> stateMachine = copycat.createStateMachine("test");
  stateMachine.open().thenRun(() -> {
    Map<String, String> map = stateMachine.createProxy(Map.class);
    map.put("foo", "Hello world!").thenRun(() -> {
      map.get("foo").thenAccept(value -> {
        System.out.println(value);
      });
    });
  });
});
```

Java 8's `CompletableFuture` framework is extremely powerful. For instance, the previous code sample could be rewritten
as so:

```java
Copycat copycat = Copycat.create(config);

copycat.open()
  .thenCompose(copycat.createStateMachine("test").open())
  .thenApply(stateMachine -> stateMachine.createProxy(Map.class))
  .thenAccept(map -> {
    map.put("foo", "Hello world!")
      .thenCompose(v -> map.get("foo"))
      .thenAccept(value -> System.out.println(value));
  });
```

### Resource clusters

In order to ensure consistency across resources, each resource performs leader election and replication separately
from the global `Copycat` cluster. Additionally, because each resource can be replicated on a separate set of nodes
than that which is defined in the global `ClusterConfig`, resources maintain a separate membership list as well.
Each resource exposes a separate `Cluster` instance that contains state regarding the resource's leader election and
membership. This `Cluster` can be used to communicate with other members in the resource's cluster.

```java
Cluster cluster = stateMachine.cluster();
```

## State machines

The most common use case for consensus algorithms is to support a strongly consistent replicated state machine.
Replicated state machines are similarly an essential feature of Copycat. Copycat provides a unique, proxy based
interface that allows users to operate on state machines either synchronously or asynchronously.

### Creating a state machine

To create a replicated state machine, you first must design an interface for the state machine states. In this case
we'll use one of Copycat's provided [collections](#collections) as an example.

States are used internally by Copycat to apply state machine commands and queries and transition between multiple
logical states. Each state within a state machine must implement a consistent explicitly defined interface. In this
case we're using the existing Java `Map` interface to define the state.

```java
public class DefaultMapState<K, V> implements Map<K, V> {
  private Map<K, V> map;

  @Initializer
  public void init(StateContext<AsyncMapState<K, V>> context) {
    map = context.get("value");
    if (map == null) {
      map = new HashMap<>();
      context.put("value", map);
    }
  }

  @Override
  public V put(K key, V value) {
    return map.put(key, value);
  }

  @Override
  public V get(K key) {
    return map.get(key);
  }

  @Override
  public V remove(K key) {
    return map.remove(key);
  }

  @Override
  public boolean containsKey(K key) {
    return map.containsKey(key);
  }

}
```

The `StateContext` can be used to store state values that Copycat will use to take snapshots of the state machine state
and compact the replicated log. For this reason, it is essential that state implementations store state in the
`StateContext`.

Once we've defined the map state, we can simply create a new map state machine via the `stateMachine` factory method.

```java
CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addStateMachineConfig("map", new StateMachineConfig()
    .withStateType(Map.class)
    .withInitialState(DefaultMapState.class));

Copycat copycat = Copycat.create(config).open().get();

StateMachine<Map<String, String>> stateMachine = copycat.createStateMachine("map").open().get();
```

When a Copycat resource - such as a state machine, log, or election - is created, the resource must be opened before
it can be used. Practically all Copycat interfaces return `CompletableFuture` instances which can either be used to
block until the result is received or receive the result asynchronously. In this case we just block by calling the
`get()` method on the `CompletableFuture`.

### Creating a state machine as a standalone service

Copycat's architecture is designed such that individual resource types can be used completely independently of others.
The state machine module can be added as a separate Maven dependency, and state machines can be created independent
of the high level `Copycat` interface which aggregates all resource types into a single component.

To use the state machine as a standalone service, simply add the `copycat-state-machine` module as a Maven dependency:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-state-machine</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

The `StateMachine` interface provides its own constructors similar to the `Copycat.create` constructors.

```java
StateMachineConfig config = new StateMachineConfig()
  .withStateType(Map.class)
  .withInitialState(DefaultMapState.class)'

StateMachine<Map<K, V>> stateMachine = StateMachine.create("my-state-machine", cluster, config).open().get();
stateMachine.submit("get", "foo").get();
```

### Configuring the state machine

State machines are configured using the `StateMachineConfig` class. This class contains vital configuration information
regarding how the state machine should handle startup and how it should apply commands.

As demonstrated in the example in the previous section, two vital components of the `StateMachineConfig` are the
*state type* and *initial state*. The state type is an *interface* which contains all of the state methods. All state
implementations must implement this interface. The initial state is the first state to which the state machine will
transition upon startup.

```java
StateMachineConfig config = new StateMachineConfig()
  .withStateType(Map.class)
  .withInitialState(DefaultMapState.class);
```

Alternatively, state machine configurations can be defined via
[Typesafe configuration](https://github.com/typesafehub/config) files. Copycat resolves state machine configurations
in the following order:
* `StateMachineConfig` - the code based state machine configuration can override all other configuration options
* User defined configuration file - the configuration resource specified when constructing a `StateMachineConfig`,
  e.g. `new StateMachineConfig("my-state-machine")`
* `state-machine` configuration resource - a `state-machine.conf`, `state-machine.json`, or `state-machine.properties`
  file on the classpath
* `state-machine-defaults.conf` - the state machine defaults configuration file
* `resource` configuration resource - a `resource.conf`, `resource.json`, or `resource.properties` file on the classpath
* `resource-defaults.conf` - the resource defaults configuration file

Additionally, configurable resources within the state machine configuration - such as the log and serializer - resolve
with a similar hierarchy of precedence.

By default, all Copycat resources use the `KryoSerializer` for serialization. This should work fine for most use cases,
but if you so desire you can configure the `Serializer` class on the `StateMachineConfig`.

```java
StateMachineConfig config = new StateMachineConfig()
  .withStateType(Map.class)
  .withInitialState(DefaultMapState.class)
  .withSerializer(MyCustomSerializer.class);
```

### Designing state machine states

State machines consist of a series of states and transitions between them. Backed by a state log, Copycat guarantees
that commands (state method calls) will be applied to the state machine in the same order on all nodes of a Copycat
cluster. Given the same commands in the same order, state machines should always arrive at the same state with the
same output (return value). That means all commands (methods) of a state machine state *must be deterministic*; commands
cannot depend on the state of shared external systems such as a database or file system.

State machines are defined by simply defining an interface. While a state machine can consist of many states (state
interface implementations), all states must implement the same interface. This allows Copycat to expose a consistent
interface regardless of the current state of the state machine.

State machine states do not have to extend any particular interface. Simply define the interface itself:

```java
public interface LockState {

  void lock();

  void unlock();

}
```

When the state machine is created, the state interface must be provided to the state machine via the
`StateMachineConfig`. Copycat uses the state interface to expose [commands](#state-machine-commands) and
[queries](#state-machine-queries) to the state machine client.

### State machine commands

State machine commands are annotated methods on the state machine state interface. Annotating a method with the
`@Command` annotation indicates that it is a stateful method call, meaning the operation that calls the method should
be persisted to the log and replicated. It is vital that *all* methods which alter the state machine's state be
identified by the `@Command` annotation. For this reason, all state machine methods are commands by default.

```java
public interface MapState<K, V> {

  @Command
  V put(K key);

}
```

### State machine queries

Queries are the counter to commands. The `@Query` annotation is used to identify state machine methods which are
purely read-only methods. *You should never annotate a method that alters the state machine state with the query
annotation.* If methods which alter the state machine state are not annotated as commands, calls to the method will
*not* be logged and replicated, and thus state will be inconsistent across replicas.

```java
public interface MapState<K, V> {

  @Query
  V get(K key);

}
```

You can specify the required consistency of queries by defining the `@Query` annotation's `consistency` argument.
The query consistency allows you to control how read operations are performed. Copycat supports three consistency
levels:
* `WEAK` - reads the state machine from the local node. This is the cheapest/fastest consistency level, but will often
  result in stale data being read
* `DEFAULT` - all reads go through the resource leader which performs consistency checks based on a lease period equal to the
  resource's heartbeat interval
* `STRONG` - all reads go through the resource leader which performs a synchronous consistency check with a majority of
  the resource's replicas before applying the query to the state machine and returning the result. This is the most
  expensive consistency level.

Query consistency defaults to `DEFAULT`

```java
public interface MapState<K, V> extends Map<K, V> {

  @Override
  @Query(consistency=Consistency.STRONG)
  V get(K key);

}
```

### The state context

Copycat's state machine provides a `StateContext` object in which state implementations should store state. The reason
for storing state in a separate object rather than in the states themselves is threefold. First, the `StateContext`
object persists throughout the lifetime of the state machine, even across states. Second, the state machine uses the
state held within the `StateContext` to take and install snapshots for log compaction. Snapshotting and log compaction
is performed automatically using the context's state; users need only store state in the context in order for snapshots
to work. Third, the state context provides a vehicle through which states can transition to other states.

To get the `StateContext` object for the state machine, simply add a method to your state annotated with the
`@Initializer` annotation.

```java
public class DefaultMap<K, V> implements Map<K, V> {
  private StateContext<AsyncMap<K, V>> context;

  @Initializer
  public void init(StateContext<AsyncMap<K, V>> context) {
    this.context = context;
  }

  public V get(K key) {
    return context.<Map<K, V>>get("the-map").get(key);
  }

}
```

### Transitioning the state machine state

To transition to another state via the `StateContext`, call the `transition` method. A perfect example of this is
a lock. This example transitions between two states - locked and unlocked - based on the current state.

```java
public class UnlockedLockState implements LockState {
  private StateContext<AsyncMap<K, V>> context;

  @Initializer
  public void init(StateContext<AsyncMap<K, V>> context) {
    this.context = context;
  }

  public void lock() {
    context.transition(new LockedLockState());
  }

  public void unlock() {
    throw new IllegalStateException("Lock is not locked");
  }

}
```

```java
public class LockedLockState implements LockState {
  private StateContext<AsyncMap<K, V>> context;

  @Initializer
  public void init(StateContext<AsyncMap<K, V>> context) {
    this.context = context;
  }

  public void lock() {
    throw new IllegalStateException("Lock is locked");
  }

  public void unlock() {
    context.transition(new UnlockedLockState());
  }

}
```

### Synchronous proxies

While Copycat's `StateMachine` interface exposes a `submit` method for submitting named commands and queries to the
state machine, it also provides a proxy based interface for operating on the state machine state interface directly.
To create a state machine proxy simply call the `createProxy` method on the `StateMachine` object, passing the proxy
interface to implement. The state machine supports both synchronous and asynchronous return values by checking whether
a given proxy interface method's return type is `CompletableFuture`.

```java
Map<String, String> proxy = stateMachine.createProxy(Map.class);
```

Once we've created a state machine proxy, we can simply call methods directly on the proxy object and Copycat will
internally submit [commands](#commands) and [queries](#queries) to the replicated state machine.

```java
proxy.put("foo", "Hello world!");
System.out.println(proxy.get("foo")); // Hello world!
```

### Asynchronous proxies

The state machine proxy feature also supports asynchronous proxies. It does this simply by checking the return type
of a given proxy method. If the proxy method's return type is `CompletableFuture` then the method will be executed
in a separate thread.

Often times asynchronous proxies require creating a separate proxy interface.

```java
public interface AsyncMap<K, V> {

  CompletableFuture<V> put(K key, V value);

  CompletableFuture<V> get(K key);

  CompletableFuture<V> remove(K key);

  CompletableFuture<Void> clear();

}
```

Aside from the return types, asynchronous proxies work the same way as synchronous proxies. To create an asynchronous
proxy simply call the `createProxy` method, passing the asynchronous proxy interface.

```java
AsyncMap<K, V> map = stateMachine.createProxy(AsyncMap.class);

map.put("foo", "Hello world!").thenRun(() -> {
  map.get("foo").thenAccept(value -> System.out.println(value)); // Hello world!
});
```

## Event logs

Event logs are eventually consistent Raft replicated logs that are designed to be compacted based on time or size.

### Creating an event log

To create an event log, call the `createEventLog` method on the `Copycat` instance.

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0", "tcp://123.456.789.1", "tcp://123.456.789.2");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addEventLogConfig("event-log", new EventLogConfig());

Copycat copycat = Copycat.create(config);

copycat.open().thenRun(() -> {
  copycat.<String>createEventLog("event-log").open().thenAccept(eventLog -> {
    eventLog.commit("Hello world!");
  });
});
```

Alternatively, an event log can be created independent of the high-level `Copycat` API by simply adding the
`copycat-event-log` module as a dependency directly:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-event-log</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

The `EventLog` interface exposes a static method for creating a standalone event log:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp:/123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog());

EventLog<String> eventLog = EventLog.create("my-event-log", cluster, config);
```

When a standalone event log is created via the `EventLog.create` static factory method, a `ClusterCoordinator` is
created and will be automatically shut down once `close` is called on the created `EventLog`.

### Event log configuration

Event logs are designed to maintain long histories of events. Copycat provides a number of configuration options for
the file logs underlying event log resources which allow users to configure both the size of the event log and the
time for which event log segments are retained.

The three most notable `Log` configuration options for event logs are the segment size, the flush interval, and
retention policies.

All Copycat logs are broken into segments internally. In order for a segment of the log to be freed (deleted from disk),
the segment must not be the last segment in the log. This means if segments were configured with the size
`Long.MAX_VALUE` then Copycat would never clean the log. The segment size and interval can be configured by using the
`withSegmentSize` and `withSegmentInterval` methods of the `Log` interface respectively:

```java
Log log = new FileLog()
  .withSegmentSize(1024 * 1024)
  .withSegmentInterval(60, TimeUnit.MINUTES);
```

The segment size dictates the maximum size of any given segment, and the interval indicates that maximum amount of
time a segment can exist before the log is rolled over to a new segment.

Users can also configure the interval at which Copycat flushes logs to disk. For event logs, typically the flush
interval does not need to be configured, allowing the `FileChannel` implementation to handle flushing to disk
internally. However, to configure the event log's flush interval use the `withFlushInterval` method:

```java
Log log = new FileLog()
  .withFlushInterval(60, TimeUnit.SECONDS);
```

Alternatively, event log configurations can be defined by
[Typesafe configuration](https://github.com/typesafehub/config) files. Copycat resolves event log configurations
in the following order:
* `EventLogConfig` - the code based event log configuration can override all other configuration options
* User defined configuration file - the configuration resource specified when constructing a `EventLogConfig`,
  e.g. `new EventLogConfig("my-event-log")`
* `event-log` configuration resource - a `event-log.conf`, `event-log.json`, or `event-log.properties`
  file on the classpath
* `event-log-defaults.conf` - the event log defaults configuration file
* `resource` configuration resource - a `resource.conf`, `resource.json`, or `resource.properties` file on the classpath
* `resource-defaults.conf` - the resource defaults configuration file

To construct an event log configuration from a custom configuration file, simply specify the configuration resource
name in the `EventLogConfig` constructor:

```java
EventLogConfig config = new EventLogConfig("my-event-log");
```

Finally, event logs support configurable retention policies. Retention policies dictate the amount of
time for which a *segment* of the log is held on disk. For instance, the `FullRetentionPolicy` keeps logs forever,
while the `TimeBasedRetentionPolicy` allows segments of the log to be deleted after a certain amount of time has passed
since the segment was created.

```java
EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog().withSegmentSize(1024 * 1024 * 32))
  .withRetentionPolicy(new SizeBasedRetentionPolicy(1024 * 1024 * 128));
```

Copycat provides the following log retention policies:
* `FullRetentionPolicy` - keeps all log segments forever
* `ZeroRetentionPolicy` - deletes segments immediately after the log has been rotated
* `TimeBasedRetentionPolicy` - deletes segments after a period of time has passed since they were *created*
* `SizeBasedRetentionPolicy` - deletes segments once the complete log grows to a certain size

Note that additional logs such as the `ChronicleLog` may provide additional configuration options related to their
specific implementation.

### Writing events to the event log

To write events to an event log, simply call the `commit(T entry)` method:

```java
eventLog.commit("Hello world!");
```

The `CompletableFuture` returned by the `commit` method will be completed once the entry has been logged and replicated
to a majority of the resource's replica set.

### Consuming events from the event log

To consume messages from the event log, register a message consumer via the `consumer` method:

```java
eventLog.consumer(entry -> System.out.println(entry));
```

## State logs
State logs are strongly consistent Raft replicated logs designed for persisting state. These logs are the basis of
Copycat's state machine and the data structures built on top of it. Whereas event logs are designed for time/size based
compaction, state logs support log compaction via snapshotting, allowing state to be persisted to the log and
replicated to other nodes in the cluster whenever necessary.

### Creating a state log

To create a state log, call the `stateLog` method on the `Copycat` instance.

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addStateLogConfig("state-log", new StateLogConfig()
    .withLog(new FileLog());

Copycat copycat = Copycat.create(config);

copycat.open().thenRun(() -> {
  copycat.<String>createStateLog("state-log").open().thenAccept(stateLog -> {
    stateLog.commit("Hello world!");
  });
});
```

The `StateLog` interface exposes a static method for creating a standalone state log:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp:/123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

StateLogConfig config = new StateLogConfig()
  .withLog(new FileLog()
    .withFlushOnWrite(true));

StateLog<String> stateLog = StateLog.create("my-state-log", cluster, config);
```

When a standalone state log is created via the `StateLog.create` static factory method, a `ClusterCoordinator` is
created and will be automatically shut down once `close` is called on the created `StateLog`.

The generic type - `T` - is the log entry type.

### State log configuration

Since state logs are designed to assist in managing state - including for Copycat's state machines and data structures -
it is strongly recommended that the state logs use persistent logs such as the `FileLog` and configure the log to
frequently flush the log to disk.

```java
StateLogConfig config = new StateLogConfig()
  .withFileLog(new FileLog()
    .withFlushOnWrite(true)
    .withSegmentSize(1024 * 1024 * 32));
```

The state log implementation uses the segment size to determine when to take snapshots. Once the log grows beyond a
single segment, the state log will take a snapshot and remove all but the first segment in the log. In order to prevent
performance issues from snapshotting too frequently, segments should not be made too small.

Alternatively, state log configurations can be defined by
[Typesafe configuration](https://github.com/typesafehub/config) files. Copycat resolves state log configurations
in the following order:
* `StateLogConfig` - the code based state log configuration can override all other configuration options
* User defined configuration file - the configuration resource specified when constructing a `StateLogConfig`,
  e.g. `new StateLogConfig("my-state-log")`
* `state-log` configuration resource - a `state-log.conf`, `state-log.json`, or `state-log.properties`
  file on the classpath
* `state-log-defaults.conf` - the state log defaults configuration file
* `resource` configuration resource - a `resource.conf`, `resource.json`, or `resource.properties` file on the classpath
* `resource-defaults.conf` - the resource defaults configuration file

To construct an event log configuration from a custom configuration file, simply specify the configuration resource
name in the `EventLogConfig` constructor:

```java
EventLogConfig config = new EventLogConfig("my-event-log");
```

### State commands

State logs work to alter state by applying log entries to registered state commands. Commands are operations which
alter the state machine state. When submitted to the state log, all commands (writes) go through the resource leader
and are logged and replicated prior to being applied to the command function.

To register a state command, use the `registerCommand` method.

```java
stateLog.registerCommand("add", entry -> data.add(entry));
```

### State queries

As you can see above state logs separate *read* operations from *write* operations, and for good reason. Write
operations require strong consistency and persistence, while read operations - which don't alter the state of the log -
can be performed without persistence. To register a read operation on the state log, use the `registerQuery` method:

```java
stateLog.registerQuery("get", index -> data.get(index));
```

Because of the flexibility of read-only operations, Copycat provides several consistency modes with which users can
trade consistency for performance and vice versa.
* `STRONG` - Guarantees strong consistency of all reads by performing reads through the leader which contacts a majority
  of nodes to verify the cluster state prior to responding to the request
* `DEFAULT` - Promotes consistency using a leader lease with which the leader assumes consistency for a period of time
  before contacting a majority of the cluster in order to check consistency
* `WEAK` - Allows stale reads from the local node

To specify the query consistency, simply pass an additional `Consistency` parameter to `registerQuery`:

```java
stateLog.registerQuery("get", index -> data.get(index), Consistency.FULL);
```

### Submitting operations to the state log

Once [commands](#state-commands) and [queries](#state-queries) have been registered on the log, commands and queries
can be submitted to the log. To submit a named operation to the log, call the `submit` method.

```java
stateLog.submit("add", "Hello world!");
```

The `CompletableFuture` returned by the `submit` method will be completed once the entry has been logged and replicated
on a majority of the resource's [active member](#active-members) replicas.

### Snapshotting

Up until this point, if the state log were allowed to run unencumbered for a long enough period of time, presumably
your machines would run out of disk space for the log. Fortunately, the Raft algorithm has a mechanism for compacting
the state log: snapshots. By registering a snapshot handler on the state log, Copycat will periodically call the handler
and replace a portion of the log with the state returned by the snapshot provider. Then, in the event of a failure the
snapshot installer will be called to repopulate the state from the last snapshot.

To register a snapshot provider on the state log, use the `snapshotWith` methods:

```java
stateLog.snapshotWith(() -> data);
```

Any snapshot provider should be accompanied by a snapshot installer as well.

```java
stateLog.installWith(data -> {
  this.data = data;
});
```

Don't worry, Copycat will handle all the complexity of persisting, loading, and replicating snapshots. If you want to
see what can be built on top of the Copycat state log, see Copycat's replicated [state machine](#state-machine)
implementation.

## Leader elections

Leader elections are perhaps the simplest feature of Copycat, but often one of the most useful as well. One of the more
significant challenges faced by many distributed systems is coordinating leadership among a set of nodes. Copycat's
simple Raft based leader election implementation allows users to elect a leader within a resource cluster and then
use Copycat's messaging and remote execution features to react on that leader election.

Note that the leader election resource type simply wraps the resource `Cluster` based `LeaderElection` API. In reality,
all Copycat resources participate in elections, and resource elections can be accessed via each respective resource's
`Cluster`.

### Creating a leader election

To create a `LeaderElection` via the `Copycat` API, add the `LeaderElectionConfig` to your `CopycatConfig`.

```java
CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addElectionConfig("election", new LeaderElectionConfig());
```

By default, leader elections use an in-memory log - `BufferedLog`. Since leader elections only use a lightweight
implementation of the Raft election algorithm to elect leaders, persistent logs are unnecessary for most use cases.

Once the leader election has been defined as a cluster resource, create a new `Copycat` instance and get the leader
election. To register a handler to be notified once a node has become leader, use the `addListener` method.

```java
Copycat copycat = Copycat.create(config);

copycat.open()
  .thenCompose(c -> c.createLeaderElection("election").open())
  .thenAccept(election -> {
    election.addListener(member -> System.out.println(member.uri() + " was elected leader!");
  });
```

Alternatively, users can use the `LeaderElection` API as a standalone service by adding the `copycat-leader-election`
module as a direct dependency:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-leader-election</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

The `LeaderElection` interface exposes a static factory method for creating standalone leader election instances:

```java
ClusterConfig cluster = new ClusterConfig()
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

LeaderElection.create("election", cluster).open().thenAccept(election -> {
  election.addListener(member -> {
    System.out.println(member.uri() + " was elected leader!");
  });
});
```

## Collections

In a partially academic effort, Copycat provides a variety of strongly consistent distributed data structures built on
top of its [state machine](#state-machines) framework.
* [AsyncMap](#asyncmap)
* [AsyncList](#asynclist)
* [AsyncSet](#asyncset)
* [AsyncMultiMap](#asyncmultimap)

Each collection provides options for configuring the consistency of read operations. To configure the consistency,
use the `setConsistency` method or `withConsistency` fluent method on the respective data structure configuration:

```java
AsyncMapConfig config = new AsyncMapConfig()
  .withConsistency(Consistency.STRONG);
```

Collections can either be created via the high-level `Copycat` API or through the respective collection constructors
themselves. Each collection interface exposes a static `create` factory method which creates a standalone instance of
the distributed collection.

To access collection factory methods directly add the `copycat-collections` module as a direct dependency:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-collections</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

To create a collection, call the static `create` factory method on the collection's interface, passing the collection
name, cluster configuration, and optional collection configuration.

```java
AsyncList.<String>create("my-list", cluster).open().thenAccept(list -> {
  list.add("Hello world!");
});
```

### AsyncMap

Copycat's map implementation uses a simple [state machine](#state-machines) to provide access to a consistent
distributed map via state machine [proxies](#synchronous-proxies).

To create a map via the `Copycat` API, use the `map` method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addMapConfig("test-map", new AsyncMapConfig()
    .withConsistency(Consistency.STRONG));

Copycat.copycat(config).open()
  .thenApply(copycat -> copycat.<String, String>createMap("test-map"))
  .thenCompose(map -> map.open())
  .thenAccept(map -> {
    map.put("foo", "Hello world!").thenRun(() -> {
      map.get("foo").thenAccept(result -> System.out.println(result));
    });
  });
```

To create a map directly, use the `AsyncMap.create` factory method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

AsyncMapConfig config = new AsyncMapConfig()
  .withConsistency(Consistency.STRONG);

AsyncMap.<String, String>create("my-map", cluster, config).open().thenAccept(map -> {
  map.put("foo", "Hello world!").thenRun(() -> {
    map.get("foo").thenAccept(result -> System.out.println(result));
  });
});
```

### AsyncList

Copycat's list implementation uses a simple [state machine](#state-machines) to provide access to a consistent
distributed list via state machine [proxies](#synchronous-proxies).

To create a list via the `Copycat` API, use the `list` method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addListConfig("test-list", new AsyncListConfig()
    .withConsistency(Consistency.STRONG));

Copycat.copycat(config).open()
  .thenApply(copycat -> copycat.<String>createList("test-list"))
  .thenCompose(list -> list.open())
  .thenAccept(list -> {
    list.add("Hello world!").thenRun(() -> {
      list.get(0).thenAccept(result -> System.out.println(result));
    });
  });
```

To create a list directly, use the `AsyncList.create` factory method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

AsyncListConfig config = new AsyncListConfig()
  .withConsistency(Consistency.STRONG);

AsyncList.<String>create("my-list", cluster, config).open().thenAccept(list -> {
  list.add("Hello world!").thenRun(() -> {
    list.get(0).thenAccept(result -> System.out.println(result));
  });
});
```

### AsyncSet

Copycat's set implementation uses a simple [state machine](#state-machines) to provide access to a consistent
distributed set via state machine [proxies](#synchronous-proxies).

To create a set via the `Copycat` API, use the `set` method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addSetConfig("test-set", new AsyncSetConfig()
    .withConsistency(Consistency.STRONG));

Copycat.copycat(config).open()
  .thenApply(copycat -> copycat.<String>createSet("test-set"))
  .thenCompose(set -> set.open())
  .thenAccept(set -> {
    set.add("Hello world!").thenRun(() -> {
      set.get(0).thenAccept(result -> System.out.println(result));
    });
  });
```

To create a set directly, use the `AsyncSet.create` factory method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

AsyncSetConfig config = new AsyncSetConfig()
  .withConsistency(Consistency.STRONG);

AsyncSet.<String>create("my-set", cluster, config).open().thenAccept(set -> {
  set.add("Hello world!").thenRun(() -> {
    set.get(0).thenAccept(result -> System.out.println(result));
  });
});
```

### AsyncMultiMap

Copycat's multimap implementation uses a simple [state machine](#state-machines) to provide access to a consistent
distributed multimap via state machine [proxies](#synchronous-proxies).

To create a multimap via the `Copycat` API, use the `multiMap` method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addMultiMapConfig("test-multimap", new AsyncMultiMapConfig()
    .withConsistency(Consistency.STRONG));

Copycat.copycat(config).open()
  .thenApply(copycat -> copycat.<String, String>createMultiMap("test-multimap"))
  .thenCompose(multiMap -> multiMap.open())
  .thenAccept(multiMap -> {
    multiMap.put("foo", "Hello world!").thenRun(() -> {
      multiMap.get("foo").thenAccept(result -> System.out.println(result));
    });
  });
```

To create a multimap directly, use the `AsyncMultiMap.create` factory method:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

AsyncMultiMapConfig config = new AsyncMultiMapConfig()
  .withConsistency(Consistency.STRONG);

AsyncMultiMap.<String, String>create("my-map", cluster, config).open().thenAccept(multiMap -> {
  multiMap.put("foo", "Hello world!").thenRun(() -> {
    multiMap.get("foo").thenAccept(result -> System.out.println(result));
  });
});
```

## The Copycat cluster

The Copycat cluster is designed not only to support leader election and replication for Copycat and its resources, but
it also serves as a general messaging utility for users as well. At the center of Copycat's cluster is the
*cluster coordinator*. The coordinator is responsible for managing communication and resources throughout the cluster.
Each `Copycat` or standalone log instance contains a single `ClusterCoordinator` through which all resources
communicate. Resource clusters wrap the central cluster coordinator to provide resource-specific leader election and
messaging. This allows Copycat to support multiple contexts within the same cluster.

### Members

The Copycat cluster consists of two different types of members, [active](#active) and [passive](#passive). Copycat's
resource behavior differs on each member of the cluster according to the member type. Additionally, each Copycat
instance maintains several versions of the cluster configuration - one global version and a cluster for each
resource. This allows for a flexible replication model where different resources within the same Copycat cluster
operate completely independently from each other in terms of leader election and log replication.

### Active members

The core of the Copycat cluster consists of a set of *active members*. Active members are required, permanent, voting
members of the Copycat cluster. Active members are defined for each Copycat instance prior to startup:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.0:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addResourceConfig(...);

Copycat copycat = Copycat.create(config);
```

By creating a Copycat instance with a local URI that is defined as an active member in the cluster configuration, Copycat
knows that this is an active member of the cluster:

```java
assert copycat.cluster().member().type() == Member.Type.ACTIVE;
```

It is important that *all members of the Copycat cluster contain identical cluster configurations*. Note that this
cluster configuration identifies *only* the core active members of the Copycat cluster, and each cluster configuration
must specify at least one active cluster member. Members that are not represented in the cluster configuration are
known as [passive members](#passive-members).

Active members are distinguished from passive members in that they fully participate in log replication via the
[Raft consensus protocol](https://raftconsensus.github.io/). This means that only active cluster members can become
leaders, and only active members can serve as synchronous replicas to cluster resources.

Since each Copycat resource contains its own separate `Cluster`, Copycat allows the resource's active membership to
differ from the global Copycat cluster membership. This means that even though the Copycat cluster may consist of five
total nodes with three active member nodes, a resource can perform synchronous replication on only three of those nodes.
In that case, the resource's Raft algorithm would run on those three nodes, and an asynchronous gossip protocol would
perform replication to the remainder of the resource's cluster.

### Passive members

Passive members are members of the Copycat cluster that do not participate in synchronous replication via the Raft
consensus protocol. Instead, Copycat uses a simple gossip protocol to asynchronously replicate committed log entries
to passive members of the cluster.

Members are defined as nodes which are not represented in the Copycat cluster configuration. Thus, starting a `Copycat`
instance or any resource with a local member URI that is not contained within the provided `ClusterConfig` indicates
that the member is passive and should thus receive replicated logs passively:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol())
  .withLocalMember("tcp://123.456.789.4:5000")
  .withMembers("tcp://123.456.789.0:5000", "tcp://123.456.789.1:5000", "tcp://123.456.789.2:5000");

CopycatConfig config = new CopycatConfig()
  .withClusterConfig(cluster)
  .addResourceConfig(...);

Copycat copycat = Copycat.create(config);
```

By simply defining a local URI that is not defined as an active member of the Copycat cluster, the Copycat instance
becomes a passive member of the cluster:

```java
assert copycat.cluster().member().type() == Member.Type.PASSIVE;
```

See below for more information on
[eventual consistency and Copycat's gossip protocol](#eventual-consistency-and-copycats-gossip-protocol).

### Member states

Throughout the lifecycle of a Copycat cluster, passive members can join and leave the cluster at will. In the case of
passive members, Copycat does not distinguish between a member leaving the cluster and a member crashing. From the
perspective of any given member in the cluster, each other member of the cluster can be in one of three states at any
given time. These three states are defined by the `Member.State` enum:
* `ALIVE` - The member is alive and available for messaging and replication
* `SUSPICIOUS` - The member is unreachable and may have crashed or left the cluster voluntarily
* `DEAD` - The member is no longer part of the cluster

Just as cluster membership can differ across resources, so too can individual member states. Each member within a
resource cluster is guaranteed to also be present in the global Copycat cluster, but members in the Copycat cluster
may not be present in each resource cluster. This is because some passive members of the Copycat cluster may not have
opened all resources, and some resources may simply not be assigned to some active members of the cluster. Only once a
resource is opened on a passive member of the cluster will it join the resource's cluster and participate in
asynchronous replication via the [gossip protocol](#eventual-consistency-and-copycats-gossip-protocol).

### Leader election

Leader election within the Copycat cluster is performed on a global and per-resource basis. Each Copycat cluster -
including the global cluster and per-resource clusters - maintains separate leader election state. Copycat provides
a simple API for listening for election events throughout the Copycat cluster.

To register an election listener for the entire Copycat cluster, use the `Copycat` instance's `Cluster`:

```java
copycat.cluster().election().addListener(result -> {
  System.out.println(result.winner() + " elected leader!");
});
```

Additionally, each resource contains the same method for accessing the resource specific cluster's election:

```java
copycat.stateMachine("my-state-machine").open().thenAccept(stateMachine -> {
  stateMachine.cluster().election().addListener(result -> {
    System.out.println(result.winner() + " elected leader!");
  });
});
```

### Direct messaging

Copycat uses a simple messaging framework to communicate between nodes in order to perform coordination, leader election,
and replication, and that framework is exposed to the user for each resource as well. Copycat's messaging framework
uses topics to route messages to specific message handlers on any given node.

To register a handler to receive messages for a topic, call the `registerHandler` method on the `LocalMember` instance.

```java
copycat.stateMachine("my-state-machine").open().thenAccept(stateMachine -> {
  stateMachine.cluster().member().registerHandler(message -> {
    return CompletableFuture.completedFuture("world!");
  });
});
```

Cluster message handlers must return a `CompletableFuture`. If the message is handled synchronously, simply return
an immediately completed future as demonstrated above. Otherwise, asynchronous calls to Copycat resources can be made
within the message handler without blocking the Copycat event loop.

To send a message to another member in the cluster, call the `send` method on any `Member` instance.

```java
copycat.stateMachine("my-state-machine").open().thenAccept(stateMachine -> {
  stateMachine.cluster().members().forEach(member -> {
    member.send("Hello").thenAccept(reply -> {
      System.out.println("Hello " + reply); // Hello world!
    });
  });
});
```

### Broadcast messaging

Copycat also allows the exposed `Cluster` to be used to broadcast messages to all members of the cluster. Note that
Copycat's broadcast functionality is *not* guaranteed, and Copycat makes no promises about either the order in which
messages will be delivered or even whether they'll be delivered at all.

Cluster-wide broadcast messages work similarly to direct messages in that they are topic based. To register a broadcast
message listener call `addBroadcastListener` on any `Cluster` instance:

```java
copycat.cluster().addBroadcastListener(message -> {
  System.out.println("Got message " + message);
});
```

To broadcast a message to all members of a cluster use the `broadcast` method on the `Cluster` instance:

```java
copycat.cluster().broadcast("Hello world!");
```

Note that broadcasts apply *only to the membership for the cluster on which the message was broadcast*. In other words,
broadcast messages will only be sent to `ALIVE` members of the cluster through which the message was sent. If you want
to broadcast a message to *all* of the members of the Copycat cluster, use the global `Cluster` which can be retrieved
through the `Copycat` instance.

### Remote execution

In addition to sending messages to other members of the cluster, Copycat also supports execution of arbitrary tasks on
remote nodes. For example, this is particularly useful for executing code on a resource's leader node once a leader has
been elected.

```java
Task<String> sayHello = () -> "Hello world!";

copycat.stateMachine("my-state-machine").open().thenAccept(stateMachine -> {
  stateMachine.cluster().election().addListener(result -> {
    result.winner().submit(sayHello).thenAccept(result -> {
      System.out.println(result); // Hello world!
    });
  });
});
```

## Protocols

Copycat's communication system provides a pluggable framework that allows the underlying message transport to be
configured based on the environment in which Copycat is deployed. Copycat uses the cluster protocol to perform all
messaging around the cluster, including for leader elections, replication, and other communication. *It is essential
that all members of the cluster configure the same protocol*, otherwise communication will fail and the cluster will
be deadlocked. Copycat provides a number of existing protocol implementations for various asynchronous frameworks.

### The local protocol

The local protocol is a special protocol that is implemented purely for testing purposes. It supports passing messages
across threads via a `ConcurrentHashMap` member registry.

The local protocol is part of `copycat-core` and thus can be simply added to any `ClusterConfig` within the need for
dependencies:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new LocalProtocol());
```

### Netty protocol

The Netty protocol is a fast [Netty](http://netty.io) based TCP protocol. To add the Netty protocol to your Maven
project, add the `copycat-netty` module to your `pom.xml`:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-netty</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

Then add the netty TCP protocol to your Copycat cluster configuration:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new NettyTcpProtocol());
```

### Vert.x protocol

The Vert.x protocol module provides several protocol implementations for [Vert.x 2](http://vertx.io), including:
* `VertxEventBusProtocol`
* `VertxTcpProtocol`
* `VertxHttpProtocol`

The Vert.x 2 protocol module provides an event bus protocol implementation for [Vert.x](http://vertx.io). To add the
Vert.x 2 protocol to your Maven project, add the `copycat-vertx` module to your `pom.xml`:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-vertx</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

Then add the Vert.x 2 event bus protocol to your Copycat cluster configuration:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new VertxEventBusProtocol("localhost", 1234));
```

You can also pass a `Vertx` instance in to the protocol:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new VertxEventBusProtocol(vertx));
```

The Vert.x module also provides a `VertxEventLoopExecutor` which executes resource callbacks on the Vert.x event loop.
To ensure callbacks are executed on the Vert.x event loop context, add the `VertxEventLoopExecutor` to the resource
configuration:

```java
EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog()
    .withRetentionPolicy(new TimeBasedRetentionPolicy(1, TimeUnit.HOURS)))
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

To configure the executor for the global Copycat cluster and instance, add an `Executor` to the `CopycatConfig`:

```java
CopycatConfig config = new CopycatConfig()
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

### Vert.x 3 protocol

The Vert.x 3 protocol module provides an event bus protocol implementation for [Vert.x 3](http://vertx.io). To add the
Vert.x 3 protocol to your Maven project, add the `copycat-vertx3` module to your `pom.xml`:

```
<dependency>
  <groupId>net.kuujo.copycat</groupId>
  <artifactId>copycat-vertx3</artifactId>
  <version>0.5.0-SNAPSHOT</version>
</dependency>
```

Then add the Vert.x 3 event bus protocol to your Copycat cluster configuration:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new VertxEventBusProtocol("localhost", 1234));
```

You can also pass a `Vertx` instance in to the protocol:

```java
ClusterConfig cluster = new ClusterConfig()
  .withProtocol(new VertxEventBusProtocol(vertx));
```

The Vert.x 3 module also provides a `VertxEventLoopExecutor` which executes resource callbacks on the Vert.x event loop.
To ensure callbacks are executed on the Vert.x event loop context, add the `VertxEventLoopExecutor` to the resource
configuration:

```java
EventLogConfig config = new EventLogConfig()
  .withLog(new FileLog()
    .withRetentionPolicy(new TimeBasedRetentionPolicy(1, TimeUnit.HOURS)))
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

To configure the executor for the global Copycat cluster and instance, add an `Executor` to the `CopycatConfig`:

```java
CopycatConfig config = new CopycatConfig()
  .withExecutor(new VertxEventLoopExecutor(vertx));
```

## Architecture

Much of the preceding documentation has revolved around the features of Copycat and how to use them.
However, many users and potential contributors may want to get a better understanding of how Copycat works. This section
will detail precisely how Copycat performs tasks like leader election and log replication via the
[Raft consensus protocol](https://raftconsensus.github.io/) and dynamic membership and failure detection via
[gossip](http://en.wikipedia.org/wiki/Gossip_protocol).

### Logs

Logs are the center of the universe in the Copycat world. The global Copycat cluster and each of the resources contained
within it are all ultimately built on a consistent, Raft replicated log. Because logs are such a central component to
Copycat, much development effort has gone into the design and implementation of each of Copycat's logs.

Copycat's logs are designed with three features in mind:
* Fast writes
* Fast reads
* Fast compaction

In order to support these three vital functions, Copycat's logs are broken into segments. Each log may consist of
several segments of a configurable size, and each segment typically has a related index. The index is a simple list
of 8-bit pairs of integers indicating the entry index and position within a given segment. This allows Copycat to
quickly locate and read entries from the log.

Additionally, each log implementation supports [log compaction](#log-compaction) on an arbitrary index.

### Strong consistency and Copycat's Raft consensus protocol

Copycat uses a unique, extensible implementation of the
[Raft distributed consensus algorithm](https://raftconsensus.github.io/) to provide strong consistency guarantees for
resource management, leader elections, and the log replication which underlies all of Copycat's resources.

Raft is the simpler implementation of only a few provable algorithms for achieving consensus in a distributed system
(the other more common implementation is [Paxos](http://en.wikipedia.org/wiki/Paxos_%28computer_science%29)).
Designed for understandability, Raft uses a mixture of RPCs and timeouts to orchestrate leader election and replication
of a distributed log. In terms of the [CAP theorem](http://en.wikipedia.org/wiki/CAP_theorem), Raft - and by
extension - Copycat is a CP system, meaning in the face of a partition, Raft and Copycat favor consistency over
availability. This makes Raft a perfect fit for storing small amounts of mission critical state.

#### Leader election

In Raft, all writes are required to go through a cluster leader. Because Raft is designed to tolerate failures, this
means the algorithm must be designed to elect a leader when an existing leader's node crashes.

In Copycat, since each resource maintains its own replicated log, leader elections are performed among the
[active member](#active-member) replicas for each resource in the cluster. This means at any given point in time,
leaders for various resources could reside on different members of the cluster. However,
[passive members](#passive-members) cannot be elected leader as they do not participate in the Raft consensus algorithm.
For instance, in a cluster with three active members and ten passive members, the leader for an event log
could be active member *A*, while a state machine's leader could be active member *C*, but neither resource's leader
could be any passive member of the cluster. Copycat's `Cluster` and its related Raft implementation are designed to act
completely independently of other `Cluster` and Raft instances within the same Copycat instance.

In order to ensure consistency, leader election in Copycat is performed largely by comparing the logs of different
members of the cluster. When a member of the Copycat cluster stops receiving heartbeats from the cluster's leader, the
member transitions into the `CANDIDATE` state and begins a new election. Elections consist of sending a `PollRequest`
to each member of the cluster requesting a vote. Upon receiving a poll request, each member will compare information
about the candidate's log with its own log and either accept or reject the candidate's request based on that
information. If the candidate's log as as up-to-date or more recent than the polled member's log, the polled member
will accept the vote request. Otherwise, the polled member will reject the vote request. Once the candidate has received
successful votes from a majority of the cluster (including itself) it transitions to `LEADER` and begins accepting
requests and replicating the resource log.

Note that since Raft requires a leader to perform writes, during the leader election period the resource will be
unavailable. However, the period of time between a node crashing and a new leader being elected is typically very
small - on the order of milliseconds or seconds.

The consistency checks present in the Raft consensus algorithm ensure that the member with the most up-to-date log in
the cluster will be elected leader. In other words, through various restrictions on how logs are replicated and how
leaders are elected, Raft guarantees that *no member with an out of date log will be elected leader*, ensuring that once
entries are committed to the Copycat cluster, they are guaranteed to remain in the log until removed either via
compaction or deletion.

#### Write replication

Once a leader is elected via the Raft consensus protocol, the Copycat cluster can begin accepting new entries for the
log. All writes to any Copycat resource are *always* directed through the resource's cluster leader, and writes are
replicated from the leader to followers. This allows writes to the resource log to be coordinated from a single
location.

Commands - write operations - can be submitted to any member of the Copycat cluster. If a command is submitted to a
member that is not the leader, the command will be forwarded to the cluster leader. When the leader receives a command,
it immediately logs the entry to its local persistent log. Once the entry has been logged the leader attempts to
synchronously replicate the entry to a majority of the [active members](#active-members) of the cluster. Once a
majority of the cluster has received the entry and written it to its log, the leader will apply the entry to its state
machine and reply with the result.

Note that writes to a Copycat resource are *not* synchronously replicated to [passive members](#passive-members).
Passive members receive only successfully committed log entries via a
[gossip protocol](#eventual-consistency-and-copycats-gossip-protocol).

Logging commands to a replicated log allows Copycat to recover from failures by simply replaying log entries. For
instance, when a state machine resource crashes and restarts, Copycat will automatically replay all of the entries
written to that state machine's log in order to rebuild the state machine state. Because all logs are built from
writes to a single leader, and because Raft guarantees that entries are replicated on a majority of the servers
prior to being applied to the state machine, all state machines within the Copycat cluster are guaranteed to receive
commands in the same order.

#### Read consistency

While Raft dictates that writes to the replicated log go through the resource's leader, reads allow for slightly more
flexibility depending on the specific use case. Copycat supports several read consistency modes.

In order to achieve strong consistency in any Raft cluster, though, queries - read operations - must go through the
cluster leader. This is because, as with any distributed system, there is always some period of time before full
consistency can be achieved. Once a leader commits a write to the replicated log and applies the entry to its state
machine, it still needs to notify followers that the entry has been committed. During that period of time, querying the
leader's state will result in fully consistent state, but reading state from followers will result in stale output.

Without additional measures, though, reading directly from the leader still does not guarantee consistency. Information
only travels at the speed of light, after all. What if the leader being queried is no longer the leader? Raft and
Copycat guard against this scenario by polling a majority of the active members of the cluster to ensure that the
leader is still who he thinks he is.

This all may sound very inefficient for reads, and it is. That's why Copycat provides some options for making
trade-offs. Copycat provides three different `Consistency` modes:
* `STRONG` - Queries go through the cluster leader. When handling a query request, the leader performs a leadership
  consistency check by polling a majority of the cluster. If any member of the cluster known of a leader with a higher
  term, the member will respond to the consistency check notifying the queried leader of the new leader, and the
  queried leader will ultimately step down. This consistency mode guarantees strong consistency for all queries
* `DEFAULT` - Queries go through the cluster leader, but the leader performs consistency checks only periodically. For
  most requests, this mode provides strong consistency. Because the default consistency mode uses leader lease
  timeouts that are less than the cluster's election timeout, only a unique set of circumstances could result in
  inconsistency when this mode is used. Specifically, the leader would theoretically need to be blocked due to log
  compaction on the most recent log segment (indicating misconfiguration) or some other long-running process while
  simultaneously a network partition occurs, another leader is elected, and the new leader accepts writes. That's a
  pretty crazy set of circumstances if you ask me :-)
* `WEAK` - Queries can be performed on any member of the cluster. When a member receives a query request, if the query's
  consistency mode is `WEAK` then the query will be immediately applied to that member's state machine. This rule
  also applies for [passive members](#passive-members).

While this functionality is not exposed by all resource APIs externally, Copycat allows read consistency to be specified
on a per-request basis via the `QueryRequest`.

#### Log compaction

One of the most important features of Raft is log compaction. While this is a feature that is often overlooked by many
Raft implementations, it is truly essential to the operation of a Raft based system in production. Over time, as
commands are written to Copycat's logs, the logs continue to grow. Log compaction is used to periodically reduce the
size of logs while maintaining the effective history of the logs.

Because Copycat's replicated log is designed to be agnostic about the structures built on top it, Copycat's Raft
implementation is effectively unopinionated about compaction. Event logs are designed to for arbitrary time or
size based compaction, while state logs and the state machines and data structures built on top of them are designed
to support compaction without losing important state information.

In order to support compaction, Copycat's logs are broken into segments. Once a log segment reaches a certain size,
the log creates a new segment and frees the written segment for compaction. Logs are compacted according to configurable
retention policies, but no policy can affect the segment that is currently being written to. This allows
historical segments to be compacted in a background thread while new segments are still being written.

For state logs, state machines, and the collections which are built on top of state machines, snapshots are used to
compact logs while preserving state. When the log grows larger than a single segment, a snapshot is taken of the state
machine state, serialized, placed at the *beginning* of the last segment, and all prior segments are permanently
deleted. This is a different pattern than is used by many other systems that support snapshots.

Because of how files are written, compacting a log segment technically requires rewriting the segment with the snapshot
at the beginning of the file. During this process, Copycat protects against data loss by creating a series of copies
of the log. If a failure occurs during compaction, persistent logs will recover gracefully by restoring the state of
the compacted segment prior to the start of compaction.

In some cases, a replica can become out of sync during the period in which the leader takes a snapshot and compacts
its log. In this case, since the leader has removed the entries prior to its snapshot it can no longer replicate those
entries to an out-of-sync follower. This is resolved by replicating the snapshot directly. By placing snapshots as
regular entries at the beginning of the log, snapshots become part of the log and thus are automatically replicated
via the Raft algorithm.

Additionally, when a resource crashes and recovers, the resource's log is replayed in order to rebuild the system's
state. By placing the snapshot at the beginning of the log, the snapshot is guaranteed to be the first entry applied
to the state machine.

### Eventual consistency and Copycat's gossip protocol

While the [active members](#active-members) of the Copycat cluster perform consistent replication of
[resources](#resources) via the [Raft consensus protocol](#strong-consistency-and-copycats-raft-consensus-protocol),
[passive members](#passive-members) receive replicated logs via a simple gossip protocol. This allows Copycat to
support clusters much larger than a standard Raft cluster by making some consistency concessions.

#### Passive membership

Every member of the Copycat cluster participates in a simple gossip protocol that aids in cluster membership detection.
When a [passive member](#passive-members) joins the Copycat cluster, it immediately begins gossiping with the
configured [active members](#active-members) of the cluster. This gossip causes the active members to add the new
passive member to the cluster configuration, which they gossip with other members of the cluster, including passive
members. All gossip is performed by periodically selecting three random members of the cluster with which to share
membership information. The gossip protocol uses a vector clock to share membership information between nodes. When
a member gossips membership information to another member, it sends member and version information for each member
of the cluster. When a member receives gossiped membership information, it updates its local membership according to
the version of each member in the vector clock. This helps prevent false positives due to out-of-date cluster membership
information.

#### Log replication

Committed entries in resource logs are replicated to passive members of the cluster via the gossip protocol. Each member
of the cluster - both active and passive members - participates in gossip-based log replication, periodically selecting
three random members with which to gossip. A vector clock on each node is used to determine which entries to replicate
to each member in the random set of members. When a member gossips with another member, it increments its local version
and sends its vector clock containing last known indexes of each other member with the gossip. When a member receives
gossip from another member, it updates its local vector clock with the received vector clock, appends replicated entries
if the entries are consistent with its local log, increments its local version, and replies with its updated vector
clock of member indexes. Tracking indexes via vector clocks helps reduce the number of duplicate entries within the
gossip replication protocol.

#### Failure detection

Just as Copycat uses gossip for membership, so too does the global cluster and resource clusters use gossip for failure
detection. The failure detection protocol is designed to reduce the risk of that false failures due to network
partitions. This is done by piggybacking failure information on to the cluster membership vector shared during gossip.
When an attempt to gossip with another member of the cluster fails for the first time, the *state* of that member in
the membership vector clock is changed to `SUSPICIOUS`. Additionally, the URI of the member that failed to communicate
with the suspicious member is added to a set of failed attempts in the membership vector clock. When a membership vector
containing a suspicious member is received by another member in the cluster, that member will immediately attempt to
gossip with the suspicious member. If gossip with the suspicious member fails, the member attempting the gossip will
again add its URI to the set of failed attempts. Once the set of failed attempts grows to a certain size (`3` by
default), the member's state will be changed `DEAD` and the member removed from the cluster.

### [User Manual](#user-manual)
