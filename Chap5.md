# Chapter 5 - Replication

Reasons for replication -
- Reduce latency (keep data geographically closer to users)
- Increase availability (the show must go on even if some parts fail)
- Increase read throughput (more machines available for read queries)

This chapter focuses on small datasets such that each machine can store a copy
of the whole dataset.

Algorithms discussed:
1. Single-Leader replication
2. Multiple-Leader replication
3. Leaderless replication

## Single-Leader Replication
Process:
- All write queries are handled by a single _leader_
- The leader sends the changes to the _followers_ via a _replication log_ or a _data stream_. Each follower updates its local copy by applying the writes from the log in the same order
- The client can query either the leader or the followers for read queries.

### Synchronous vs Asynchronous Replication
In _Synchronous Replication_, the leader sends the write query to be updated to the followers. After the follower confirms that it has received the write request, the leader gives an OK to the client.
In _Asynchronous Replication_, the leader doesn't wait for the follower confirmation. It sends the write result and then gives an OK to the client.

Advantage of synchronous - The followers will always have an up-to-date copy of the data in the leader. So if the leader fails, the follower still has the latest data.

Disadvantage of synchronous - If a follower lags or doesn't respond, the leader can't send an OK to the client. Leader is also  blocked from further write queries until the follower is available again.

To handle this disadvantage, only 1 follower is made synchronous (instead of all of them) and the others are asynchronous. If the synchronous follower is unavailable/slow, one of the other asynchronous followers can be made synchronous. This ensures that at least 2 nodes have the complete dataset at all times.

### Adding new followers
To add new follower while the database is still receiving write queries:
1. Snapshot the database (without halting write queries) and note the snapshot position in the replication log.
2. Copy snapshot to the new follower.
3. Add all transactions after the snapshot position to the follower.
4. After the follower has _caught up_ to the leader, continue processing changes as usual.

### Handling Failures
- Follower failure: Catch-up recovery

The follower log will have the last transaction it processed before failure. The follower can connect to the leader and request all changes after that point. It can then _catch up_ to the leader and then start processes as usual.

- Leader Failure : Failover

Steps needed:
1. Detect that leader has failed
2. Determine new leader
3. Reconfigure system to use new leader

### Implementing Replication logs
#### Statement-based replication
Write the whole SQL statement (insert, update, delete) to the log. The follower will then run the SQL statement as-is.

Disadvantages -
- Nondeterministic queries (NOW(), RAND()) will have different results in different followers
- If statement has auto-incrementing columns or depends on exists data in any way (such as filters), the SQL statements must be executed in the exact same order. This is difficult when there are multiple concurrent transactions
- Statements with side-effects (triggers, stored procs, UDFs) will have different effects in each follower unless it is absolutely deterministic.

#### Write-ahead log shipping
An append-only byte-sequence log which contains all the writes. When the follower processes this log, it creates a copy of the exact same data structure as found on the leader. Disadvantage - This is a byte-level log so replication is tied to storage engine version. If database changes storage format, nodes with different versions can't replicate each other.

#### Logical (row-based) log shipping
Use different log formats for replication and for storage engine for decoupling.
- For Inserts, contain new values of all columns
- For Deletes, contain info to uniquely identify deleted row. Primary key if possible, otherwise all column values
- For Updates, contain info to uniquely identify deleted row and new values of updated columns
These records can also be sent to external applications e.g. _change data capture_

#### Trigger based replication
Above approaches are handled by the database. But what if only some data should be replicated? Or what if a different database is used? In that case, application level code is required for replication. A trigger can call custom application code which is used when data change occurs. This code will provide application logic for replication.

### Problems with replication lag
_Eventual Consistency_ : During asynchronous replication: It is possible that the replica has not received/processed all changes from the leader. In this case, the replica and the leader may have different data available. If a client queries the two nodes at the same time, it may get different data due to _replication lag_. But eventually, the replica will catch up to the leader and both nodes will have the same data. Example problems follow -

#### Reading your own writes - Facebook comment lag
In Facebook: If a user comments on a post, and then reads the post from a lagging follower, the new comment will not be visible which will make them unhappy. This requires _read-after-write consistency_ i.e. if a user does a read query, they must see the data they have written themselves. This only guarantees that the user can see their own comment, it doesn't guarantee that they'll see comments by other users on the post.

Techniques:
- If a user wants to read something they modified, read it from the leader, otherwise use follower. Database has to track whether something has been modified or not. e.g. User should view their own FB profile from the leader, other profiles from follower.
- For one minute after the last write, read all queries from the leader. Or track replication lag on followers and only followers with less than 1 minute lag should handle queries.
- Client should track last update time. System should only provide read queries from replicas with a processed_time more than the client update time.

#### Monotonic reads - Disappearing comments
In Facebook: A user queries a follower, sees 5 comments on a post, refreshes the page. This refresh is served by a second follower which lags behind the first follower and so only 3 comments are returned. In this case, the user feels that 2 comments have disappeared. _Monotonic reads_ : The user will not read older data after having read newer data in a previous query.

Techniques:
- A user should always read from the same replica, different users can read from different replicas.

#### Consistent Prefix Reads - Convo appearing in reverse order
Discworld Terry Pratchett Mrs. Cake - A precog who answers questions 10 seconds before they are asked.
_Consistent Prefix Reads_ : If a sequence of related writes happens in a certain order, anyone reading those writes must see them appear in the same order.

## Multi-Leader Replication
In Single-Leader replication, if leader is down or network problems between user and leader, no writes can be performed. Extension - allow multiple leaders which can handle writes

### Use Cases
#### Multi-datacenter operation
Have 1 leader in each datacenter. Between datacenters, the leaders replicate changes to other leaders.

Benefits compared to single-leader config:
- Performance - In single-leader, user had to write to leader which could be geographically far away. Now, users can write data to closest leader/datacenter resulting in low latency + better performance for the user. The changes can then be replicated to other datacenters 'behind the scene'
- Tolerance of datacenter outages - Now, each datacenter can operate independently. The failed datacenter can catch up after it comes online.
- Tolerance of network problems - Temporary public internet problems don't prevent writes happening within a datacenter.

Disadvantage - Same data can be modified in 2 datacenters at the same time resulting in a conflict

#### Clients with offline operation
e.g. Calendar, Google Keep. User can still modify events/notes even without a network connection. So each client / mobile device can act as a local leader. This leader can then sync and replicate changes to online Google datacenters when internet connection is available again.

#### Real-Time collaborative editing
e.g. Google Docs. When a user makes a change, it is applied to their local leader (document in web browser/application) which is then synchronized with server and other online users. This synchronization can happen after very small changes (e.g. a single keystroke) for faster collaboration.

### Handling Write Conflicts
Conflict Avoidance - All writes related to a given record always go through the same leader. This ensures conflict is avoided (as long as the datacenter doesn't go down or have other problems)

Convergence - Conflicts must be resolved in a _convergent_ way i.e. all replicas must arrive at the same value after all changes have been replicated. Achieving convergence:
- _Last Write Wins_ : The write with the most recent timestamp wins. This leads to data loss for older timestamps
- Replica priority : Writes from higher replicas win. This leads to data loss for lower replicas
- Merge values : Merge strings together by sorting + concatenation.
- Custom Conflict Resolution Logic : Record conflicts and write application code which resolves conflict later (maybe by asking user for input)

Custom Conflict Resolution Logic
- On Write : When a conflict is detected while writing data, call the conflict handler
- On Read : When database tries to read data and a conflict is detected, call the conflict handler

### Multi-Leader Replication Topologies
_Replication topology_ - `Communication path along which writes are propagated from one to another`
1 - Circular Topology - Each node writes to another node in a circle
2 - Star Topology - 1 root node sends writes to all other nodes
3 - All-to-all Topology - Each node sends writes to all other nodes
