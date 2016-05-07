IPFS Cluster specification
==========================

Related issue: https://github.com/ipfs/notes/issues/58

Motivation
----------

[IPFS](https://ipfs.io) is a distributed file system based on Merkle DAGs, where objects are referenced by hashes corresponding to their contents.

IPFS mechanism for data permanence is called "pinning". When an IPFS nodes "pins" an object, that object is locally stored and retained, thus making the node a permanent seeder for the object. Non-pinned object's survival depends on a number of factors, including their popularity (how often they are read from the network) or the available size on the nodes (unpopular objects are likely to be forgotten at some point by their seeders if they are not pinned).

Pinning, however, is an explicit operation that is local to the node. This raises the question of how to ensure the survival of objects in an IPFS network in a general manner, so that the network can be used as **permanent storage**. IPFS Cluster aims to essentially address this problem.

Features
--------

IPFS Cluster should provide the following features:

- Replicated: In an IPFS cluster with n = 3 replication factor, there should *eventually* be 3 full copies of every object. Each copy should be stored in a different node.
- Reliable: In the event of nodes dissapearing, IPFS cluster ensures that under-replicated objects are picked up by other nodes.
- Auto-balanced: It is possible to think of several balancing strategies (disk-space/bandwith/location/network-topology based) but disk-space balance seems like a good general-case starting point.
- Fully HA and/or distributed: IPFS is itself distributed. Any system built on top should pay close attention to prevent the introduction of single points of failure.
- Easy to use: strive for simplicty and sensible defaults.
- Transparent: Users can keep using the IPFS network just like before.
- Auditable: who has what (now and in previous point of time)

Implementation
--------------

### Dummy

A single IPFS cluster "director" node accepts requests to persist a Hash. The director node keeps a table with IPFS nodes in the cluster, and the current state (disk space etc) and properties. The director then select N nodes and request that they pin these hashes. The director has a map (Hash -> Nodes) that keeps track of what is pinned where.

The director "pings" the nodes regularly to check that they are alive. When they are not, the director removes them from the (Hash -> Nodes) tables and selects a new node to restore the replication factor. When a node re-appears, they are requested to unpin all the content (as it was replicated somewhere else in their absence).

Pros:
  - Simple
  - Auditable
Cons:
  - No HA
  - Not scalable
  

### Master-slave

The IPFS cluster "master" node accepts requests to persist a Hash. It then selects one of the available cluster "slaves" (say by round-robin) and assigns the task of replicating the hash. It keeps track of which slaves handle which Hashes and receives heartbeats from the slaves. When a slave goes down, it is able to reassign the replication tasks to a different one.

Each of the slaves has a list of hashes they are rensposible to replicate and a table to track which IPFS node has a replica. Slaves monitor IPFS nodes on which they are replicating data and check that they are alive (as well as collect metrics). When an IPFS node goes down, the slave instruct others IPFS nodes to pin the underreplicated content. When they come back, they are requested to unpin the hashes they were responsible for before.

A secondary master in inactive state can be placed next to the main one. The main one copies its state to it. When the main one goes down, the secondary one takes it's place.

Pros:
  - More HA
  - Scales better
Cons:
  - Master is still a bottleneck
  - First coordination problems (slaves try to do things at the same time in the same node)

### Multi-master with RAFT

The IPFS cluster counts with several "masters" in the sense that any of them can receive requests to persist a Hash. This requires coordination between the different masters. The masters need to share a state object which tracks which master is taking care of which hashes.

Coordination is performed by electing an "orchestrator" which ensures that the state object is correctly synced and distributed to every master. This can be done by using RAFT (modifications to the state being Raft's log entries).

When a master A receives a request to persist a Hash X, it politely asks the "orchestrator" (RAFT's leader) to validate this. The orchestrator is the authoritative source for "who is persisting what", and this information is shared with all the other masters. Once the log message stating that A takes care of X is persisted, A takes the necessary steps to pin the hash X in the IPFS nodes of the cluster.

If an IPFS cluster node (follower) goes down, the orchestrator can assign its share to a different master. If an IPFS node goes down, the masters which have objects in that node can ensure that the objects are replicated somewhere else.

If the orchestrator goes down, RAFT ensures the election of a new leader and the former leader is treated like any other dissapeared follower.

Pros:
  - Looks more like proper HA
  - Tasks are balanced
  - Coordination overhead is small
Cons:
  - RAFT scalability (Multiraft?)




Links
-----

* [Accrual Failure Detector](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf) (Cassandra)
* CockroachDB: Multiraft: raft with consensus groups: https://www.cockroachlabs.com/blog/scaling-raft/
