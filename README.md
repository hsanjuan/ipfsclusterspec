IPFS Cluster specification
==========================

This document is just gathering some random and naïve thoughts on how to implement an IPFS cluster solution. It's work in progress and written in a way I understand it, without caring much about if it is comprehensible to a different mind than mine.

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

Each of the slaves has a list of hashes they are rensposible to replicate and a table to track which IPFS node has a replica. Slaves monitor IPFS nodes on which they are replicating data and check that they are alive (as well as collect metrics). When an IPFS node goes down, the slave instruct other IPFS nodes to pin the underreplicated content. When they come back, they are requested to unpin the content they were responsible for before.

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

When a master A receives a request to persist a Hash X, it politely asks the "orchestrator" (RAFT's leader) to select someone to perform the task. The orchestrator is the authoritative source for "who is persisting what", and this information is shared with all the other masters. Once the log message stating that a slave takes care of X is persisted, that slave takes the necessary steps to pin the hash X in the IPFS nodes of the cluster.

If an IPFS cluster node (follower) goes down, the orchestrator can assign its share to a different master. If an IPFS node goes down, the masters which have objects in that node can ensure that the objects are replicated somewhere else.

If the orchestrator goes down, RAFT ensures the election of a new leader and the former leader is treated like any other dissapeared follower.

The orchestrator can additionally use IPFS to store and persist the state regularly, allowing the auditing and recovery of the system in cases of full disaster.

Pros:
  - Looks more like proper HA
  - Tasks are balanced
  - Coordination overhead is small
  - Auditable with replicated log
Cons:
  - RAFT scalability (Multiraft?)

### Distributed with Conflict-free Replicated Data Types (solution 1)

CRDTs allow to build masterless/distributed architectures by using data types which eventually converge to the same values. Since we are distributed, we cannot assign the tasks to persis a hash to specific, but rather, only share if and by which IPFS node a hash is persisted.

This requires a map [Hash -> ORSet(IPFSNode)]. When an IPFS cluster node receives a request to persist a hash, it selects the IPFS nodes to pin it and modifies the distributed ORSet for that map.
If a request to persist a hash is received on two different IPFS cluster nodes at the same time, it may be that the hash is over-replicated. This is an issue to deal with in the future.

If an IPFS Cluster node goes down, nothing happens. However when it comes back again it will need to rebuild the Map from somewhere. This somewhere can be IPFS itself, assuming it knows some of its peers and that those peers regularly store the state or updates (linked list?).

If an IPFS node goes down, someone has to realize about it (problem: everyone needs to watch every hash) and persist the hashes of that node somewhere else. On large IPFS Clusters with many IPFS Cluster nodes it is possible that this task is attempted by several nodes at the same time, causing, again, undersirable side effects like over-replication.

One big problem is that every IPFS Cluster node needs to take care of too many things (since we have no orchestrator to divide and assign tasks): watching IPFS nodes for failures, maintaining the full map with all the hashes up to date, merging on conflicts and making sure it stays consistent in the face of network failures of channel issues.


Pros:
 - Fully distributed
 - No need for consensus protocols
 - Easy auto-scaling
Cons:
 - The states we are handling are big in big IPFS clusters, thus an operation-based replication would be better but,
 - Operation-based replication is not very resilient against failures of the channels
 - Optimized state-based replication (using deltas) might be a compromise, but still requires sending full states from time to time
 - Coordination overhead, specially as it grows big.
 - Eventual consistence and quirks of CRDTs (add-wins in ORSet etc) create room for funny side-effects that need to be dealt with separately (over-replication at the very least)


Open questions
--------------

- I took the assumption that IPFS stores full objects per node (rather than blocks of a single object)
- JBenet's virtualized IPFS nodes representing an IPFS vnode which shards the data among a bunch of real IPFS nodes is a different problem. IPFS vnodes conceptually simplify the layout of IPFS clusters used for massive amounts of data. In their simplest form, they are a proxy to a subcluster. In their not not-so-simple form, they raise many questions:
  - What happens if the virtual node goes down? If we don't want a whole subcluster to suffer, we're back to same problem: need HA and consensus.
  - Nice IPFS properties (like DSHT distance metrics) are lost and a vNode makes the cluster behave like any old style key-value permanent storage (S3). In that sense, it may actually be interesting to think of a vIPFS node as a node with an IPFS API but underlying storage based off something completely different (say S3, Hadoop, cassandra).
  - vNodes can become a read-bottleneck
  - Taking the multi-master IPFS cluster approach, we can think of making each master a vNode, and the IPFS cluster underneath would just be partitioned under each master. Probably this is an idea that can be developed later.
- Pinsets/"request-to-persist-a-hash" etc are to be defined but it is not an issue to have meta-objects about such things backed-up in IPFS in the form @jbenet suggests.

Links
-----

* [Accrual Failure Detector](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf) (Cassandra)
* CockroachDB: Multiraft: raft with consensus groups: https://www.cockroachlabs.com/blog/scaling-raft/
* [A comprehensive study of convergent and commutative replicated data types](http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf)
