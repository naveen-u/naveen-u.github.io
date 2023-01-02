---
title: Apache ZooKeeper Basics
date: 2022-02-14 21:37:41 +0530
categories: [Technical]
tags: [technical, distributed-computing] # TAG names should always be lowercase
summary: Notes on Apache ZooKeeper and how it helps solve some co-ordination problems in distributed systems.
preamble: Notes originally written for an internal tech talk I delivered at work.
math: true
---

## Introduction

ZooKeeper is an open-source distributed co-ordination service for distributed applications. It is essentially a distributed, linearizable, hierarchical file-system which is highly available, fault-tolerant, and fast.

## How It Works

### Ensemble

ZooKeeper is itself a distributed system. It follows a simple client-server model where clients are nodes that make use of the service, and servers are nodes that provide the service. A collection of ZooKeeper servers constitute an ensemble.

### Quorum

A quorum is a strict majority of nodes ($n/2 + 1$) in an ensemble. If a quorum of nodes are not available (up and running) in an ensemble, the ZooKeeper service is non-functional. This means that an ensemble of $n$ nodes can tolerate up to $(n-1)/2$ failures. Hence, both a 3-node ensemble and a 4-node ensemble can only handle a single-node failure. For this reason, ensembles are generally made up of an odd number of nodes (adding an extra node to make it even adds no extra tolerance).

### Leader and Followers

A ZooKeeper ensemble always consists of one leader and zero or more followers. As soon as an ensemble starts, the nodes elect a leader by consensus (read more about this [here](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html#sc_leaderElection)). Every election of a new leader is followed by a synchronization phase before any new changes are accepted. If at any point, a leader disconnects, a new leader is elected from the remaining servers as long as quorum is met. The newly elected leader will be the most up-to-date out of all the candidates. If at any point, the total number of nodes in the ensemble is less than quorum, the leader (if it exists) will step down and no new leader will be elected. This is to prevent any new writes to the system which can cause the ensemble to become inconsistent (refer [Read & Write Operations](#read--write-operations)).

### Znodes

ZooKeeper allows distributed processes to coordinate with each other through a shared (replicated) hierarchical namespace which is organized similarly to a standard file system. In the hierarchy, each node (called a znode in ZooKeeper nomenclature) can contain some (or no) data, and can also have zero or more children. All data is kept in-memory to make ZooKeeper extremely performant. However, this also means that the amount of data that can be stored in a znode is limited (1MB).

![ZooKeeper Namespaces](/assets/img/posts/ZooKeeper/zknamespace.jpg)
_ZooKeeper Namespaces: ZooKeeper's hierarchical namespace. Image courtesy of the [Apache ZooKeeper docs](https://zookeeper.apache.org/doc/r3.1.2/zookeeperOver.html)._

Znodes can be one of:

- Persistent: A persistent znode is kept alive even after the client session that created it disconnects. By default, all znodes are persistent unless specified otherwise.
- Ephemeral: Ephemeral znodes are active only as long as the client session that created it is connected. They get deleted automatically when the creator client disconnects. Due to this, ephemeral znodes cannot have children (since a znode can only be deleted if it has no child znodes).

Znodes can also be sequential (while being either persistent/ephemeral). When a new znode is created as a sequential znode, ZooKeeper sets the path of the znode by suffixing a 10 digit sequence number to the original name. For example, if a znode with path `/foo` is created as a sequential znode, ZooKeeper will change the path to `/foo0000000001` and set the next sequence number to `0000000002`.

### Client Session

At any given time, each client is connected to exactly one server. Each server can handle a large number of client connections at the same time. Each client periodically pings the server it's connected to to let it know that it's alive and connected. The server in question responds with an acknowledgment of the ping, indicating the server is alive as well. When the client doesn’t receive an acknowledgment from the server within the specified time, the client connects to another server in the ensemble, and the client session is transparently transferred over to the new ZooKeeper server. The new server that the client connects to is guaranteed to be at least as up-to-date as the previous server (refer point 3 under [Guarantees](#guarantees)).

### Read & Write Operations

In an ensemble, any node can read data, but all writes are orchestrated by the leader. When a client wants to write some data to a znode, it sends a request to whichever server it’s connected to. The server forwards this request to the leader. The leader receives write requests from followers and publishes changes to the other servers in an ordered fashion (refer point 1 under [Guarantees](#guarantees)). The followers executes the change and sends an acknowledgement back to the leader. The write is considered to be committed (or successful) if and only if a quorum of acknowledgements is received (refer point 2 under [Guarantees](#guarantees)). If a server doesn’t acknowledge the change within a set time (`syncLimit x tickTime`, read more about it [here](https://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_configuration)), it will be dropped from the ensemble (refer point 5 under [Guarantees](#guarantees)).

### Operations

ZooKeeper is very simple in terms of the operations it allows:

- `create`: creates a node at a location in the tree
- `delete`: deletes a node
- `exists`: tests if a node exists at a location
- `get data`: reads the data from a node
- `set data`: writes data to a node
- `get children`: retrieves a list of children of a node
- `sync`: waits for data to be propagated

Apart from these, ZooKeeper also supports the concept of _watches_. Clients can set a watch on a znode. A watch will be triggered and removed when the znode changes. When a watch is triggered, the client receives a packet saying that the znode has changed.

These simple operations can be used to build powerful co-ordination systems thanks to ZooKeeper's guarantees.

### Guarantees

ZooKeeper provides a set of guarantees:

1. Sequential Consistency: Updates from a client will be applied in the order that they were sent.
2. Atomicity: Updates either succeed or fail. No partial results.
3. Single System Image: A client will see the same view of the service regardless of the server that it connects to. i.e., a client will never see an older view of the system even if the client fails over to a different server with the same session.
4. Reliability: Once an update has been applied, it will persist from that time forward until a client overwrites the update.
5. Timeliness: The clients view of the system is guaranteed to be up-to-date within a certain time bound.

## Common Usages

Even though the ZooKeeper API is fairly simple, the guarantees it provides enable clients to build reliable co-ordination systems. Some commonly used recipes can be found [here](https://zookeeper.apache.org/doc/current/recipes.html). Below, we talk about the two specific use-cases of ZooKeeper.

### Leader Elections

#### What is it?

Leader election is the simple idea of giving one thing in a distributed system (say, one pod in a K8s deployment) some special powers. Those special powers could include the ability to assign work, the ability to modify a piece of data, or even the responsibility of handling all requests in the system. Note that here we're talking about leader election in a client system and not the ZooKeeper ensemble.

#### How can we do it with ZooKeeper?

When a client wants to contend leadership, it:

1. Creates a sequential ephemeral znode in a pre-determined path, say `/leader/election`` with a unique name (a UUID, perhaps).

   - Since the znode is sequential, ZooKeeper assigns it a sequence number that’s greater than any of the previously appended children of `/leader/election`.
   - An ephemeral node gets deleted as soon as the client session that created it disconnects. This allows a new leader to be automatically selected even if the current leader dies.

2. Gets the children of the leader election parent node.
   - This returns all of the nodes currently in contention for leadership.
3. If the znode created in step 1 has the lowest sequence number, it assumes leadership, and can proceed with leader-only duties.
   - Note how the client with the _smallest_ znode is the leader, and not the largest. This is so that, even if new clients join later on, the leader doesn’t keep changing. The leader only changes if the current leader disconnects (causing the ephemeral node to be deleted) or when it relinquishes leadership (by deleting the node itself).
4. Otherwise, the client calls `exists` with a watch on the znode immediately preceding its znode.
   - Each client only watches it’s preceding znode instead of the current leader, or the parent. This is to avoid a herd effect. If all the clients watched the leader, all of them would be notified when the leader dies. All of them would then send a request to ZooKeeper to fetch the updated list of children, and all of them would re-set their watches. This would cause unnecessary load on the ZooKeeper ensemble.
5. If `exists` returns a falsy value, go back to step 2. Otherwise, wait for the watch notification.
6. If the client gets a notification that the znode it was watching has been deleted, go back to step 2.
   - It’s important for the client that received the notification to check the list of children again. The client immediately preceding it in the queue could have disconnected (say, due to a network error) before it even got a chance to become the leader. Just because a client got a watch notification does not necessarily mean it’s the new leader.

If a client wishes to relinquish leadership or drop out of contention, it can simply delete the znode it created in step 1.

### Distributed Locks

#### What is it?

The purpose of a lock is to ensure that among several nodes that might try to do the same piece of work, only one actually does it at a given time. Taking a lock prevents concurrent processes from messing up the state of the system, or reading inconsistent data.

#### How can we do it with ZooKeeper?

The recipe for distributed locking is similar to the leader election recipe. In fact, leader election is in principle just a long-lived lock that’s never released (unless the client fails).
When a client wishes to obtain a lock, it:

1. Creates a sequential, ephemeral znode at a pre-defined lock node location with a unique name (a UUID, perhaps).
2. Gets the children of the lock node.
3. If the znode created in step 1 has the lowest sequence number, the client has the lock and is finished acquiring it.
4. Otherwise, the client calls `exists` with a watch on the znode immediately preceding its znode.
5. If `exists` returns a falsy value, go back to step 2. Otherwise, wait for a notification on the watch that was set.
6. If a notification is received on the watch, go to step 2.

When a client wishes to release a lock, it simply deletes the znode it created in step 1.

### Additional Notes

#### Create failures

If a recoverable error occurs while creating an ephemeral node, the client should call `getChildren()` and check for a node containing the unique ID used in the path name. This handles the case of the `create()` succeeding on the server but the server crashing before returning the name of the new node.

## Apache Curator

> _Curator [noun]: a keeper or custodian of a museum or other collection - A ZooKeeper keeper._

[Apache Curator](https://curator.apache.org/docs/about/) is a Java/JVM client library for ZooKeeper. It includes a high-level API framework and utilities to make using ZooKeeper much easier and more reliable. It also includes recipes for common use cases and extensions such as service discovery and a Java 8 asynchronous DSL.
It includes recipes for various common ZooKeeper usages including (but not limited to):

- Leader elections
- Locking
- Counters
- Caches
- Queues
