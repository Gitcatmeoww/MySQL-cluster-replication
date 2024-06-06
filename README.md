# MySQL Cluster & Replication

## Cluster

### What is MySQL Cluster

- **MySQL Server**: This is the primary software that handles client connections and threads, parses, and optimizes queries. Typically, it runs as a _single process_ on a _single machine_.

- **Deficiency in High Availability**: For applications requiring high availability — such as telecom, web services, and financial services — a single running MySQL server is not sufficient.

  - For example, if a bank's database server goes down, it could disrupt all transactions, causing significant issues for customers and the bank itself. Therefore, it's essential to have a redundant infrastructure that can quickly become accessible to the application in case of failure.

- **MySQL Cluster**: MySQL Cluster addresses this issue by utilizing a _distributed architecture_, making it more suitable for large-scale and high-concurrency application scenarios compared to a traditional MySQL server.
  - In a MySQL Cluster, multiple nodes work together to ensure the database is always available and can handle a high volume of transactions simultaneously.

### Two Types of MySQL Cluster

#### InnoDB Cluster

##### Overview

- An InnoDB Cluster consists of at least three MySQL Server instances, and it offers built-in automatic failover and recovery capabilities.
  ![alt text](img/innodb_cluster_overview.png)

##### Components

1. **MySQL Shell**: A command-line tool for easily configuring and administering a group of at least three MySQL server instances to function as an InnoDB Cluster.

2. **MySQL Server Instances**: Each MySQL server instance in an InnoDB Cluster runs MySQL Group Replication, which provides the mechanism to replicate data within an InnoDB Cluster, with _built-in failover_.

3. **Group Replication**: Ensures data consistency and fault tolerance by replicating data across multiple instances.

4. **MySQL Router**: Directs traffic to the correct server, providing load balancing and transparent failover.

#### NDB Cluster

##### Overview

- NDB Cluster is a technology that enables clustering of in-memory databases in a _shared-nothing system_.

  - The shared-nothing architecture enables the system to work with very inexpensive hardware, and with a minimum of specific requirements for hardware or software.

- An NDB Cluster consists of a set of computers, known as _hosts_, each running one or more processes. These processes are known as _nodes_.
  ![alt text](img/ndb_cluster_overview.png)

##### Components

1. **SQL Nodes**: Provide the SQL interface for clients to interact with the cluster.

   - In the case of NDB Cluster, an SQL node is a traditional MySQL server that uses the `NDBCLUSTER` storage engine.

2. **Management Nodes**: Manage the configuration and monitoring the other nodes within the NDB Cluster.

   - E.g., providing configuration data, starting and stopping nodes, and running backups.

3. **Data Nodes**: Store the actual data and handle data processing.

#### Comparison & Use Cases

|                  | InnoDB Cluster                                                                                                                                      | NDB Cluster                                                                                                                                                                                       |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **🤗 Pros**      | - Easy to set up and manage using MySQL Shell.<br>- Provides automatic failover and recovery.<br>- Uses the familiar InnoDB storage engine.         | - High performance with low latency, optimized for real-time applications.<br>- Highly scalable, capable of handling large numbers of nodes.<br>- Robust fault tolerance with automatic failover. |
| **🤧 Cons**      | - May introduce some latency due to Group Replication.<br>- Typically scales to a smaller number of nodes compared to NDB Cluster.                  | - More complex to set up and manage.<br>- Requires more hardware resources.                                                                                                                       |
| **🪝 Use Cases** | - Web applications needing high availability and strong transactional support.<br>- E-commerce sites that require robust data integrity and uptime. | - Telecom systems processing real-time data.<br>- Financial services requiring low-latency transactions.<br>- Large-scale web services needing high throughput.                                   |

## Replication

### Overview

- **Problem**: Among database instances in the cluster, how do we keep data synchronized within the same topology?
- **Solution**: Replication, which involves writing events that modify data or data structure to a log on a source server. Replica servers can then read the events from the log on the source and replay them. This is generally an asynchronous process. At a high level, replication is a simple three-part process:
  1. **Binary Log on Source**: The source records changes to its data in its binary log as “binary log events.”
  2. **Relay Log on Replica**: The replica copies the source’s binary log events to its own local relay log.
  3. **Replaying Events on Replica**: The replica replays the events in the relay log, applying the changes to its own data.
     ![alt text](img/replication_solution.png)

### Replication Under the Hood

#### Binary Log Formats

1. **Statement-based replication**: Record the query that changed the data on the source.

   - 🟢 Simple and compact.
   - 🔴 Has problems with nondeterministic queries.

2. **Row-based replication**: Write events to the binary log that indicate how the row has changed.

   - 🟢 Deterministic, provides the safest method of replicating data.
   - 🔴 Writing the events for row data changes for every row affected can increase the size of the binary log dramatically.

3. **Mixed method**: Use statement-based format as the default and only switching to row-based when it needs to.

#### Global Transaction Identifiers

- **Problem**: If the source has crashed and you had to rebuild the data from a backup, how could you reattach your replica if the binary logs started over again?
- **Solution**: Global transaction identifiers (GTIDs) for tracking replication positions.
  - With GTIDs, every transaction that a source server commits is assigned a unique identifier.

### Types of Replication

- Traditional MySQL Replication provides a simple source to replica approach to replication. The source is the primary, and there are one or more replicas, which are secondaries. The source applies transactions, commits them and then they are later (thus asynchronously) sent to the replicas to be either re-executed (in statement-based replication) or applied (in row-based replication). It is a **shared-nothing system**, where all servers have a full copy of the data by default.

#### Asynchronous Replication

- The source server _does not wait_ for replicas to confirm receipt of data before committing the transaction. This means there could be a slight delay (lag) between when data is updated on the source and when it is updated on the replicas.
- **Pros**: Higher performance and lower latency for the source server because it does not have to wait for replicas.
- **Cons**: There is a risk of data loss if the source server fails before the data is replicated.
  ![alt text](img/async_replication.png)

#### Semisynchronous Replication

- Every transaction that source commits must be acknowledged as received by _at least_ one replica.
  - The acknowledgment confirms that the replica received it and successfully wrote it to its own relay log (but not necessarily applied it to the local data).
  - If no replicas acknowledge the transaction during the time frame, MySQL reverts to its standard asynchronous replication.
- Since each transaction must wait on the response from other nodes, this feature adds additional latency to every transaction that the server does.
- **Use Case**: In the event of a network partition, an isolated source isn’t still writing data while partitioned from its replicas.
  ![alt text](img/semisync_replication.png)

#### Synchronous Replication

- The source server waits for _all replicas_ to confirm receipt of data before committing the transaction. This ensures that data is consistent across all replicas at all times.
- **Pros**: Strong consistency, as all replicas are guaranteed to have the same data as the source.
- **Cons**: Can introduce latency because the source must wait for acknowledgments from all replicas.

#### Group Replication

- The replication group is a set of servers that each have their own entire copy of the data (a shared-nothing replication scheme), and interact with each other through message passing.
- Each server in the group may execute transactions independently at any time.
- However, all read-write transactions commit only after they have been approved by the group.
  - In other words, for any read-write transaction the group needs to decide whether it commits or not, so the commit operation is not a unilateral decision from the originating server.
  - Read-only transactions need no coordination within the group and commit immediately.
- **Certification**: Detect conflicts between transactions that execute concurrently on different servers.
  - Conflict detection is carried out at row level: If two concurrent transactions, that executed on different servers, update the same row, then there is a conflict.
  - Distributed first commit wins: The conflict resolution procedure states that the transaction that was ordered first commits on all servers, and the transaction ordered second aborts, and is therefore rolled back on the originating server and dropped by the other servers in the group.
    ![alt text](img/group_replication.png)

### Recommended Replication Topologies

#### Active/Passive

- Direct all reads and writes to a single source server.
- Maintain a small number of passive replicas that do not actively serve any application traffic.
- **Use Case**: Don’t want to worry about replication lag.
  - Since all reads go to the source, prevent any read-after-write problems that an application may not tolerate.
- **Caveats**: Explicitly binding read scaling to the capacity of a single server.

#### Active/Read Pool

- Direct all writes to the source. Reads can be sent to either the source server or the read pool.
- **Use Case**: A read pool allows to scale reads horizontally for read-intensive applications.
- **Caveats**: Application must have some tolerance for stale reads. You will never be able to guarantee that a write you complete on the source has already been replicated to a replica.
