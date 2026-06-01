# Cap Theorem

## Consistency
> Atomic, or linearizable, consistency is the condition expected by most web services today. Under this consistency guarantee, there must exist a total order on all operations such that each operation looks as if it were completed at a single instant. This is equivalent to requiring requests of the distributed shared memory to act as if they were executing on a single node, responding to operations one at a time.

a system is consistent if an update is applied to all relevant nodes at the same logical time. Among other things, this means that standard database replication is not strongly consistent. As anyone whose read replicas have drifted from the master knows, special logic must be introduced to handle replication lag.

That said, consistency which is both instantaneous and global is impossible. The universe simply does not permit it. So the goal here is to push the time resolutions at which the consistency breaks down to a point where we no longer notice it.

## Availability
> For a distributed system to be continuously available, every request received by a non-failing node in the system must result in a response. That is, any algorithm used by the service must eventually terminate … When qualified by the need for partition tolerance, this can be seen as a strong definition of availability: even when severe network failures occur, every request must terminate.

Despite the notion of “100% uptime as much as possible,” there are limits to availability. If you have a single piece of data on five nodes and all five nodes die, that data is gone and any request which required it in order to be processed cannot be handled.

## Partition Tolerance
> In order to model partition tolerance, the network will be allowed to lose arbitrarily many messages sent from one node to another. When a network is partitioned, all messages sent from nodes in one component of the partition to nodes in another component are lost. (And any pattern of message loss can be modeled as a temporary partition separating the communicating nodes at the exact instant the message is lost.)

Some systems cannot be partitioned. Single-node systems (e.g., a monolithic Oracle server with no replication) are incapable of experiencing a network partition. But practically speaking these are rare; add remote clients to the monolithic Oracle server and you get a distributed system which can experience a network partition (e.g., the Oracle server becomes unavailable).

Network partitions aren’t limited to dropped packets: a crashed server can be thought of as a network partition. The failed node is effectively the only member of its partition component, and thus all messages to it are “lost” (i.e., they are not processed by the node due to its failure). Handling a crashed machine counts as partition-tolerance. (Update: I was wrong about this part. See this for more.) (N.B.: A node which has gone offline is actually the easiest sort of failure to deal with—you’re assured that the dead node is not giving incorrect responses to another component of your system.)

For a distributed (i.e., multi-node) system to not require partition-tolerance it would have to run on a network which is guaranteed to never drop messages (or even deliver them late) and whose nodes are guaranteed to never die. You and I do not work with these types of systems because they don’t exist.

## Extras

* CP systems -> data is consistent between all nodes, and maintains partition tolerance (preventing data desync) by becoming unavailable when a node goes down.

* AP systems -> nodes remain online even if they can't communicate with each other and will resync data once the partition is resolved, but you aren't guaranteed that all nodes will have the same data (either during or after the partition)

* CA systems -> data is consistent between all nodes - as long as all nodes are online - and you can read/write from any node and be sure that the data is the same, but if you ever develop a partition between nodes, the data will be out of sync (and won't re-sync once the partition is resolved).

* CA system dont exists:

> As a thought experiment, imagine a distributed system which keeps track of a single piece of data using three nodes — A, B, C. and which claims to be both consistent and available in the face of network partitions. Misfortune strikes, and that system is partitioned into two components: [A, B], [C]. In this state, a write request arrives at node C to update the single piece of data.

That node only has two options:

1. Accept the write, knowing that neither A nor B will know about this new data until the partition heals.
2. Refuse the write, knowing that the client might not be able to contact A or B until the partition heals.

You either choose availability (Door #1) or you choose consistency (Door #2). You cannot choose both.

To claim to do so is claiming either that the system operates on a single node (and is therefore not distributed) or that an update applied to a node in one component of a network partition will also be applied to another node in a different partition component magically.

This is, as you might imagine, rarely true.

Google cloud spanner promises in-effect CA, not theoretical: https://cloud.google.com/blog/products/databases/inside-cloud-spanner-and-the-cap-theorem





