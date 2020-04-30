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

This also needs additional bookkeeping - The system has to track which keys are hot and need this randint() annd which keys do not need it.

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
