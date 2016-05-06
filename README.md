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
- Auto-balanced: The cluster storage should be used *evenly*. It is possible to think of several balancing strategies (disk-space/bandwith/location/network-topology based) but disk-space balance seems like a good general-case starting point.
- Fully HA and/or distributed: IPFS is itself distributed. Any system built on top should pay close attention to prevent the introduction of single points of failure.
- Easy to use: strive for simplicty and sensible defaults.
- Transparent: Users can keep using the IPFS network just like before.
- Auditable: who has what (now and in previous point of time)


Links
-----

* [Accrual Failure Detector](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf) (Cassandra)
* 
