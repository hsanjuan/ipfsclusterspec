IPFS Cluster specification
==========================

This document is just gathering some random and thoughts on how to implement an IPFS cluster solution. It's work in progress.

Related issue: https://github.com/ipfs/notes/issues/58

Motivation
----------

[IPFS](https://ipfs.io) is a distributed file system based on Merkle DAGs, where objects are referenced by hashes corresponding to their contents.

IPFS mechanism for data permanence is called "pinning". When an IPFS nodes "pins" an object, that object is locally stored and retained, thus making the node a permanent seeder for the object. Pinning, however, is an explicit operation that is local to the node. The survival of non-pinned objects depends on a number of factors, including their popularity (how often they are read from the network) or the available size on the nodes (unpopular objects are likely to be forgotten at some point by the network).

 This raises the question of how to ensure the survival of objects in an IPFS network in a general manner, so that the network can be used as **permanent storage**. IPFS Cluster aims to essentially address this problem.

Features
--------

IPFS Cluster should provide the following features:

- IPFS Cluster is built on top of IPFS
- Replication: In an IPFS cluster every object should *eventually* be replicated to an indicated factor. Each copy should be stored in a different node.
- Reliable: In the event of nodes dissapearing, IPFS cluster ensures that under-replicated objects are picked up by other nodes.
- Auto-balanced: Content is distributed evenly across the network. Distribution may be based on several factors (disk-space/bandwith/location/network-topology). Disk-space seems like a good general-case starting point.
- HA and/or distributed: IPFS Cluster must not have a single points of failure.
- Scalable: IPFS Cluster can scale along with the number of content it has to oversee.
- Easy to use: strive for simplicty, sensible defaults and easy deployment.
- Transparent: Users can keep using the IPFS network just like before.
- Auditable: there is a track record featuring who has what.

Implementation thoughts
-----------------------

IPFS Cluster can be implemented using a cluster of supervisor nodes (*IPFSCluster nodes* from now on), which coordinate their actions using an IPFS-based implementation of the RAFT consensus algorithm (Consul, etcd, RethinkDB).

IPFS Cluster nodes provide an endpoint `Persist(Hash, ...)` which instructs the IPFS Cluster to ensure that an IPFS object (or tree) is replicated and pinned in several IPFS network members (*IPFS nodes* from now on).

The IPFSCluster distributes across its IPFSCluster nodes, (with the help of a RAFT's elected leader), the task of replicating sets of Hashes. The IPFSCluster nodes monitor the IPFS nodes which they have engaged in replication and ensure that, when they become unavailable, the content is replicated somewhere else.

The IPFSCluster leader tracks every request to persist a Hash, assigns it to one of the IPFSCluster nodes and watches them. In case of unavailability of an IPFSCluster node, the content assigned to it is transferred to one of the other members. The IPFS Cluster leader is also in charge of performing rebalancing of the assignments when needed.

Let's look a bit more in detail to the implementation;

### Raft on IPFS

RAFT is a weak (majority-based) consensus protocol which allows a set of participating nodes to agree on the contents of a distributed log. Once a log is marked as commited, it ensures that such log entry has been agreed upon by the members of the cluster and will eventually be picked up by every of them.

It is possible to implement RAFT on IPFS with the help of IPNS, albeit not as fast as traditional implementations. IPFS-RAFT heartbeat intervals will depend on how IPNS behaves to queries/changes. That said, very fast hearbeats and small log updates offer not so many advantage over slow hearbeats and larger log updates as we will see.

Lets get into details:

* In RAFT nodes receive or send messages (messages being log entries, log entry proposals, heartbeats, votes etc).
* In IPFS-Raft, receiving reading the hash pointed by the IPNS entry of the node you want to receive from (so polling at regular intervals is required).
* In IPFS-Raft, sending means updating the node's own IPNS hash to point to the information it wants to send.

With this we have a way for RAFT cluster members to communicate. With that, we can think of a first approach to how the RAFT messages would look like.

For example, the RAFT leader would like to persist a type, so it sends a message proposing to persist a log entry

```
Mode: leader
Type: proposal
UniqueID: 123456
ProposeLog: <ipfs_hashA> # points to the new log entry
Log: <ipfs_hashB> # points to the current commited log entry
```

And followers would answer:

```
Mode: follower
Type: ack
UniqueID: 123456
AckLog: <ipfs_hashA>
Log: <ipfs_hashB>
<...other custom fields...>
```

If the master sees a majority of followers have provided an ack it will send the message:

```
Mode: master
Type: commit
UniqueID: 123458
Log: <ipfs_hashA>
```

And the followers will publish:

```
Mode: follower
Type: ack
UniqueID: 123458
AckLog: <ipfs_hashA>
Log: <ipfs_hashA>
<...other custom fields...> # Note that a follower can use the ack messages to send back any other custom information to the master
```

The fact that a follower publishes a message with the same UniqueID as the master acknowledges that the follower has seen the master's message. The IPNS systems provides message-signing out-of-the-box, so as long as the set of cluster members is known by all the participants, the messages are auto-signed by using IPNS to distribute them.

This mechanism, with very similar messages, can be used for hearbeats when no log updates happen.

In the case when a leader does not get majority to commit a message, it can keep sending requests with different UniqueIDs. The leader also keeps an eye on the information published by the rest of cluster members. If a new candidate for leadership shows up, then the master needs to step down and an election needs to happen. Without entering in the election procedure part of the algorithm, we can think of messages in the form:

```
Mode: candidate
Round: 1
```

```
Mode: voter
Round: 1
Vote: node02
```

With a working IPFS-RAFT implementation, we can assume that the IPFSCluster nodes have agree on an IPFS Hash. In practice, this hash is the head of a log. The format of the contents of this log hash are application dependent and, for IPFSCluster, are detailed in the next section.

### IPFSCluster Log format

IPFS cluster nodes run side by side with regular IPFS nodes, which they use to transmit messages and share a common distributed log, agreed upon using RAFT as stated above.

This log is a linked list. On a first approach, each entry of the list would look like:

```
parent: <hash>
assignments:
  node00:
    - hash1:
        operation: persist
        recursive: yes
        where: [ipfsnode_33, ipfsnode_65, ipfsnode_44] # fully optional. Here to easen handover in rebalancing.
	- hash2:
        operation: unpersist
        recursive: yes
	- hash3:
        operation: persist
        recursive: no
	- ...
  node01:
    - hash60:
        operation: forget
  node03:
    - hash60:
        operation: persist
hash_list:
  - node00: <ipfs_hash>
  - node01: <ipfs_hash>
  - ...
```

The message provides instructions for each node in the IPFS cluster to tend to the persistence of a given hash (recursively or not), or to the unpersistance (unpinning) or others (if that is supported). Adding a replication factor per object would be just another parameter. It is the cluster leader's task to divide each request-to-persist-a-hash among the followers (or itself). It can do this, for example, just by round-robin.

In turn, each IPFS cluster member keeps a `hash_list` object which provides a list of all the hashes it is tending to and in which IPFS nodes they are pinned:

```
# hash_list
<hash1>:
  where: [ipfsnode_id1, ipfsnode_id33, ipfsnode_id50]
  ...
<hash2>:
  where: [ipfsnode_id11, ipfsnode_id23, ipfsnode_id99]
...
```

This list has several purposes: first, to keep a what-is-persisted-where relation, which can be used if the original maintainer goes offline and handed over to a new responsible node(s). Second, to test if the "assignments" made to a node are being honored. Third, to keep an account of who is persisting what at any given point in time. I am inclined to only include in this list objects which are fully persisted (they have been pinned and fully transfered to IPFS node). The comparison between the assignments and the `hash_list` should provide an idea of how much an IPFS Cluster node is lagging behind.

The `hash_list` is itself persisted on IPFS and its latest-version-hash is made available to the IPFSCluster leader as part of the RAFT ACKs from the cluster members. The leader, in turn, includes the hashes in the log entries so they are retrievable if needed.

At this point, each IPFS cluster node knows which hashes he has assigned and has the task to select a suitable IPFS nodes to pin the content. How this selection is performed is TBD, but it should support different strategies (along the line of Cassandra's snitching for example). In the most basic form, it could be based on available space known IPFS nodes.

### Re-balancing

#### IPFS cluster member offline

When an IPFS cluster node loses the connection to the leader, or in the case of network partition, the RAFT protocol ensures that:
- Nodes without majority cannot become leaders or take over the log
- Followers without leader will not do any changes

So the cluster stays in a consistent state.

In the case a cluster member goes offline, the leader (or the new elected leader) knows:
- Which hashes were assigned to that member (from the log and the `hash_list`)
- Where are those hashes pinned (from the `hash_list`)

The leader can simpy re-assign those hashes to the rest of the members of the cluster with the next message. If the lost member comes back, the leader can re-balance again the hashes assigned to each member.

#### IPFS node offline

In the case where an IPFS node, stops responding (see section about monitoring IPFS nodes), the IPFS cluster members which were persisting hashes to that node need to re-persist any content (which would be now underreplicated). They can select new IPFS nodes and pin the content to them, and consequently update the `hash_list` to reflect the new locations.

The question of a node coming back which has pinned content which has in turned been re-pinned somewhere else is open.

### Monitoring IPFS nodes

This section is still very open. In it's simplest way, IPFS Cluster nodes can just regularly check if nodes are responding to commands (getHasList etc), if their ports are open and so on.

Because of the nature of IPFS, it would be better to use an [Accrual Failure Detector](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf) (used by Cassandra), which dynamically adjusts to the behaviour of the monitored nodes.

### Scaling IPFS Cluster

There are several questions about scalability with this implementation which are open:

**How many hashes can IPFS Cluster manage/persist**: This depends on the performance of IPFS-RAFT and the capacity of the underlying IPFS network to handle the data structures used by IPFS Cluster (for example, the hash lists which will grow). A way of attacking the problem might be to use a separate ad-hoc IPFS network only for IPFSCluster communication, which is not subject to the eventualities of the larger network (for example nodes randomly coming and going from public IPFS networks).

Another problem in this regard is the number of IPFS nodes that an IPFSCluster node can monitor in order to detect unavailability and underreplication. Increasing the number of IPFS Cluster members to balance this task implies to increase the number of participants in the RAFT consensus, which is usually kept low. Network/hash partitions and multiraft (https://www.cockroachlabs.com/blog/scaling-raft/) might offer some relief on this regard.

IPFS Virtual nodes as proposed by @jbenet might offer a solution to ever-growing IPFS networks, by encapsulating many nodes under a single one, and effectively easening IPFSCluster tasks.

**How many hashes can IPFS Cluster ingest per second**: The second question is basically how hard can IPFSCluster be hammered with requests to persist new content. While RAFT would be rather slow, there is nothing preventing the leader to append many new requests to the new log entries until a proposal is formalized. This may however generate large log entries, which add further delays since they need to be persisted on IPFS too. Network bandwidth and general health of the IPFS network (and speed of IPNS) come into play for this quesion.


### Other open questions

- I took the assumption that IPFS stores full objects per node (rather than blocks of a single object). Hope this is right.
- @jbenet's *Virtual IPFS nodes* representing an IPFS vnode which shards the data among a bunch of real IPFS nodes is a different problem. IPFS vnodes conceptually simplify the layout of IPFS clusters used for massive amounts of data. In their simplest form, they are a proxy to a subcluster. In their not not-so-simple form, they need to be themselves HA, make sure that nice properties of IPFS are not lost (DSHT distance metrics) etc. monitor the subcluster for healthiness and so on. There is an obvious question to this: could an IPFSCluster node be an IPFS vNode at the same time? The hashes could then be persisted to its own subcluster. Moreover, having vNodes would probably help with some scaling issues. So this is an idea worth exploring but for the moment I have left it out.
- I don't mention "Pinsets", but they are just an IPFSCluster `persist(...)` request with multiple hashes so it is easy to include the concept.
- Also, what the `persist(...)` endpoint is exactly is not very defined. IPFSCluster could take IPFShashes pointing to PinSets, or could offer a REST endpoint or a CLI etc.
- I have considered @kubuxu's proposal of using CRDTs to implement IPFSCluster. In fact, I would love to use CRDTs to do it, but I have been unable to come up with a solution which behaves well enough. The problems I find are mostly when things go wrong and IPFS or IPFSCluster nodes dissapear. A bunch of distributed nodes without any coordination other than a eventually-consistent-shared state need to start taking decision on how to re-persist and/or rebalance the cluster. This leaves, imho, too much space for inconsistent decisions, which, additionally, cannot be easily reverted (because that means more decisions). That said, I'm very new to CRDTs and I'm fully open to discuss about it.

