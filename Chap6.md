# Chapter 6 - Partitioning

Normally, each record belongs to exactly 1 partition. This allows for scalable queries - queries which touch only one partition can be sent to that processor. So queries can be scaled by adding more nodes.

Partitioning and Replication - Copies of each partition are stored on multiple nodes for fault tolerance.

## Partitioning of Key-Value Data

Partition Skew - If some partitions have a lot more data than other partitions, queries can take more time. The query has to wait till the bigger partition is done processing. This is called _skew_. The partition with the larger load can be called the _host spot_

For even distribution, the data could be assigned to nodes randomly but then it would be difficult to query items. If we wanted to find a particular record, we'd have to query all the nodes in parallel.

### Partitioning by Key Range
Consider an encyclopedia containing multiple books. Book 1 can contain 'A', Book 2 - 'B', Book 10 - 'U'-'Z'. Now, if we want to find a particular key, we can pick the relevant book and then search for it. Deciding the boundaries of each book (letter-wise breakdown) can be done manually or automatically.

Since the keys are sorted alphabetically in each partition, we can easily perform _range queries_ on them by getting all related records together. This can be useful for timestamp keys (year-month-date) and if we want to get all 2020-Apr records at once.

A downside of this partitioning is that hot spots can happen. eg If we write current real-time data frequently, the current day partition will have lots of writes while other partitions will sit idle.

### Partitioning by Hash of Key
`A good hash function takes skewed data and makes it uniformly distributed`. Assign each partition a range of _hashes_ and every key with a hash in that range is stored in that partition. Partition boundaries can be evenly spaced or chosen pseudorandomly (_consistent hashing_).

But since adjacent keys are scattered across partitions, we can not do efficient range queries anymore. Some databases uses _compound primary keys_ where the first column is hashed and the later columns are partitioned by range. e.g. If we had real-time sensor data from different sensors, the sensors would be hashed and the timestamp data would be partitioned by range. So we could perform range queries on sensors now.

## Skewed workloads and relieving hot spots
Concatenate a random integer (say 1-100) before/after each key to distribute it more evenly between 100 partitions.

This results in additional work for reading queries, since each query will now have to get data from 100 partitions and combine it together. So instead of doing randint() for all keys, only do it for the hot keys.

This also needs additional bookkeeping - The system has to track which keys are hot and need this randint() and which keys do not need it.

## Secondary Indexes
For this section, assume that we are operating a website for selling used cars. Each listing has a `document ID` and each car has `color` and `make`. The database is partitioned by document ID. The user should be able to query by `color` and `make`.

### Document-based partitioning - Local Index - Scatter/gather
Each partition maintains a secondary index containing the IDs for each `color` and `make` value present in documents inside it. It doesn't contain any information about listing in other partitions. So it is a _local index_. Whenever a new red car is added to that partition, it updates the secondary index automatically.

_Scatter/gather_ : If we want to find all red cars across the database, we need to _scatter_ our query across all partitions and then _gather_ the results together. This leads to delays when one partition takes too long to reply - 'tail latency amplification'.

### Term-based partitioning - Global Index - Secondary Index
A global index covers data from all partitions but even this index will need to be partitioned. So this global index is partitioned on _term values_.

This allows for efficient reads since a read query only needs to read from the partition containing the term.

However, this slows down write queries because now all the relevant terms have to be updated (which may be spread across different partitions). Since this is a slow process, writes to secondary indexes are often asynchronous to avoid blocking main query.

## Rebalancing Partitions
Over time, - query throughput increases, - dataset size increases, - machines fail so we need to move data between nodes. This process is called rebalancing.

Requirements:
- After rebalancing, load (reads, writes, data size) should be shared fairly between nodes
- While rebalancing, reads and writes should continue happening
- Data transfer should be minimized to reduce time taken and network I/O

### Rebalancing Strategies
Preferred way - Divide all possible hashes into ranges and assign each range to a partition
#### Wrong way - hash mod N
_hash(key) % N_ (N: number of nodes) does not work because if N changes, we'll have to shuffle keys between nodes. eg. with N=5, key=12 goes to partition number 2 (12%5=2) but if N increases to 6, the key has to be moved to partition number 0 (12%6=0). This frequent movement makes rebalancing expensive.

#### Better way - Fixed number of partitions
Assign several partitions to each node e.g if we have 10 nodes, create 1,000 partitions in the beginning and assign ~100 partitions to each node.

- Now if a new node is added, assign a few of those 100 partitions from each node to the new partition. So each node now has ~90 partitions.
- If a node is removed, do the reverse. Distribute its partitions among the existing nodes.
While this addition/deletion of nodes is going on, the old partition-to-node mapping can be used. Only entire partitions are moved, the key-to-partition mapping and number of partitions doesn't change. Only partition-to-node mapping changes.

Improvement - More partitions can be assigned to more powerful nodes to account for hardware differences.

#### Dynamic Partitioning
Maintain a partition-node mapping. If the size of a partition exceeds a threshold, split it into 2 partitions and send it to another node. If the size becomes too small, combine it with an adjacent partition.

Benefit - If data size is small, only a few partitions are required resulting in small overhead. If data size increases, number of partitions can increase as needed.

#### Partitioning proportionally to nodes
Assign a _fixed_ number of partitions to each node. As dataset size increases, number of partitions stays constant, only partition size increases.

If a new node is added, it randomly chooses a fixed number of partitions and takes ownership of half their data.

### Rebalancing : Automatic or Manual?
`fully automatic rebalancing - the system decides automatically when to move partitions from one node to another, without any administrator interaction`

`fully manual - the assignment of partitions to nodes is explicitly configured by an administrator, and only changes when the administrator explicitly reconfigures it`

Rebalancing is expensive and needs lot of network bandwidth to shuffle data. Fully automatic is unpredictable and cause unintended domino effects. e.g if a node temporarily slows down, other nodes will think it is dead and start the rebalancing process. i.e. They will try to move load away from the slow node. This data shuffling will cause additional load on the slow node and other nodes - making the situation worse.

So, it is good to have a human confirm things before rebalancing. This avoids surprises.

### Request Routing
How does the client know which node to query for a given key? 3 possible ways:
1. Round-robin - Client asks any node in network. If the node has the data, it sends a response directly. Otherwise, it forwards the request to correct node and sends the reply back to client
2. Routing tier - This service knows the key-partition-node mapping and redirects request to correct node
3. Client-side - Client stays aware of the partition-node mapping and directly contacts the correct node

Then how does the object in all 3 cases (node, routing service or client) know the correct mapping? This job is done by a coordination service such as **ZooKeeper**.

ZooKeeper maintains the _authoritative_  partition-node mapping. Other objects subscribe to ZooKeeper for changes so that they are kept up-to-date.
