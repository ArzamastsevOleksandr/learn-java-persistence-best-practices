# Notes on Java Persistence

* The fear of database portability can lead to avoiding highly effective features just because they are not interchangeable across various database systems. In reality, it is more common to end up with a sluggish database layer than having to port an already running system to a new database solution.
* [JPA](link) and [Hibernate](link) were never meant to substitute SQL, and [native queries](link) are unavoidable in any non-trivial enterprise application. While JPA makes it possible to abstract DML statements and common entity retrieval queries, when it comes to reading and processing data, nothing can beat native SQL.
* [JPQL (Java Persistence Querying Language)](link) abstracts the common SQL syntax that is supported by most relation databases. Because of this, JPQL cannot take advantage of [Window Functions](link), [Common Table Expressions](link), [Derived tables](link) or [PIVOT](link).
* Just like [Criteria API](link) protects against [SQL injection attacks](link) when generating entity queries dynamically, [JOOQ](link) offers the same safety guarantee when building native SQL statements.

### Performance and scaling
* In application performance management, the two most important metrics are **response time** and **throughput**. The lower the response time, the more responsive an application becomes. **Response time** is, therefore, the **_measure of performance_**. Scaling is about maintaining low response times while increasing system load, so **throughput** is the **_measure of scalability_**.
* The transaction response time is measured as the sum of: 
  * the database connection acquisition time
  * the time it takes to send all database statements over the wire
  * the execution time for all incoming statements
  * the time it takes for sending the result sets back to the database client
  * the time the transaction is idle due to application-level computations prior to releasing the database connection
* Throughput can be calculated as the number of transactions executed within a given time interval:
  * X = transactionCount / time
* So, by lowering the transaction execution time, the system can accommodate more requests.
* Ideally, if the system were scaling linearly, adding more database connections would yield a proportional throughput increase. Due to [contention](https://www.cockroachlabs.com/blog/what-is-database-contention/) on database resources and the cost of maintaining coherency across multiple concurrent database sessions, the relative throughput gain follows a curve instead of a straight line.
* Every connection requires a [TCP](link) socket from the client (application) to the server (database).
* A look into database system internals reveals the tight dependency on CPU, Memory, and Disk resources. Because I/O operations are costly, the database uses a buffer pool to map into memory the underlying data and index pages. Changes are first applied in memory and flushed to disk in batches to achieve better write performance.
* High-throughput database applications experience contention on CPU, Memory, Disk, and Locks. When all the database server resources are in use, adding more workload only increases contention, therefore lowering throughput.
* Lowering response time not only makes the application more responsive, but it can also increase throughput. However, response time alone is not sufficient in a highly concurrent environment. To maintain a fixed upper bound response time, the system capacity must increase relative to the incoming request throughput. Adding more resources can improve scalability up to a certain point, beyond which the capacity gain starts dropping.

#### Master-Slave replication
* For enterprise systems where the *read/write* ratio is high, a **Master-Slave** replication scheme is suitable for increasing availability. 
* The *Master* is the system of record and the only node accepting writes. All changes recorded by the *Master* node are replayed onto *Slaves* as well. A binary replication uses the *Master* node while a statement-based replication replays on the *Slave* machines the exact statements executed on Master. 
* Asynchronous replication is very common, especially when there are more *Slave* nodes to update.
* The *Slave* nodes are **eventual consistent** as they might lag behind the *Master*. In case the *Master* node crashes, a cluster-wide voting process must elect the new *Master* from the list of all available *Slaves*.
* The asynchronous replication topology is also referred as **_warm standby_** because the election process does not happen instantaneously.
* Most database systems allow one synchronous *Slave* node, at the price of increasing transaction response time (the *Master* has to block waiting for the synchronous *Slave* node to acknowledge the update). In case of *Master* node failure, the automatic failover mechanism can promote the synchronous *Slave* node to become the new *Master*.
* Having one synchronous *Slave* allows the system to ensure data consistency in case of *Master* node failures since the synchronous *Slave* is an exact copy of the *Master*. The synchronous *Master-Slave* replication is also called a **_hot standby_** topology because the synchronous *Slave* is readily available for replacing the *Master* node.
* When only asynchronous *Slave* nodes are available, the newly elected *Slave* node might lag behind the failed *Master*, in which case **_consistency and durability_** are traded for **_lower latencies and higher throughput_**.
* Aside from eliminating the **single point of failure**, database replication can also increase transaction throughput without affecting response time. In a *Master-Slave* topology, the *Slave* nodes can accept read-only transactions, therefore routing read traffic away from the *Master* node.
* The *Slave* nodes increase the available read-only connections and reduce *Master* node resource contention, which can also lower read-write transaction response time. If the *Master* node can no longer keep up with the ever-increasing read-write traffic, a Multi-Master replication might be a better alternative.


#### Multi-Master replication
* In a Multi-Master replication scheme, all nodes are equal and can accept both read-only and read-write transactions. Splitting the load among multiple nodes can only increase transaction throughput and reduce response time as well.
* However, because distributed systems are all about trade-offs, ensuring data consistency is challenging in a Multi-Master replication scheme because there is no longer a single source of truth. The same data can be modified concurrently on separate nodes, so there is a possibility of conflicting updates. The replication scheme can either avoid conflicts or it can detect them and apply an automatic conflict resolution algorithm.
* To avoid conflicts, the [two-phase commit](link) protocol can be used to enlist all participating nodes in one distributed transaction. This design allows all nodes to be in sync at all time, at the cost of increasing transaction response time (by slowing down write operations).
* If nodes are separated by a WAN (Wide Area Network), synchronization latencies may increase dramatically. If one node is no longer reachable, the synchronization will fail, and the transaction will roll back on all Masters.
* Although avoiding conflicts is better from a data consistency perspective, synchronous replication might incur high transaction response times. On the other hand, at the price of having to resolve update conflicts, asynchronous replication can provide better throughput. The asynchronous Multi-Master replication requires a conflict detection and an automatic conflict resolution algorithm. When a conflict is detected, the automatic resolution tries to merge the two conflicting branches, and, in case it fails, manual intervention is required.


#### Sharding
* When data size grows beyond the overall capacity of a replicated multi-node environment, splitting data becomes unavoidable. Sharding means distributing data across multiple nodes, so each instance contains only a subset of the overall data.
* Traditionally, relational databases have offered *horizontal partitioning* to distribute data across multiple tables within the same database server. As opposed to *horizontal partitioning*, **sharding** requires a distributed system topology so that data is spread across multiple machines.
* Each shard must be self-contained because a user transaction can only use data from within a single shard. Joining across shards is usually prohibited because the cost of distributed locking and the networking overhead would lead to long transaction response times.
* By reducing data size per node, indexes also require less space, and they can better fit into main memory. With less data to query, the transaction response time can also get shorter too.
* Each data center can serve a dedicated geographical region, so the load is balanced across geographical areas. Not all tables need to be partitioned across shards, smaller size ones being duplicated on each partition. To keep the shards in sync, an asynchronous replication mechanism can be employed.
* In the quest for increasing system capacity, sharding is usually a last resort strategy, employed after exhausting all other available options, such as:
  * optimizing the data layer to deliver lower transaction response times
  * scaling each replicated node to a cost-effective configuration
  * adding more replicated nodes until synchronization latencies start dropping below an acceptable threshold


### JDBC Connection Management
* The JDBC (Java Database Connectivity) API provides a common interface to communicate with the DB. All networking logic and DB-specific communication protocol are hidden behind the vendor-independent JDBC API. The `java.sql.Driver` is the entry point for interacting with the JDBC API, defining the implementation version details and providing access to a DB connection.
* JDBC defines 4 driver types, of which the *Type 4* (communication protocol implemented solely in Java) is the preferred alternative being easier to setup and debug.
* To communicate with the DB, a Java program first obtains a `java.sql.Connection`. Although the `java.sql.Driver` is the actual DB connection provider, it is more convenient to use the `java.sql.DriverManager` since it can also resolve the JDBC driver associated with the current database connection URL.
* Every time the `getConnection()` method is called, the `DriverManager` requests a new physical connection from the underlying `Driver`.
* In a typical enterprise application, the user *request throughput* is greater than the available *DB connection capacity*. As long as the *connection acquisition time* is tolerable, the requests can wait for a DB connection to become available. The middle layer acts as a *DB connection buffer* that can mitigate user request traffic spikes by increasing request response time, without depleting DB connections or discarding incoming traffic. Because the intermediate layer manages connections, the application server can also monitor connection usage and provide statistics to the operations team. For this reason, instead of serving physical connections, the application server provides only logical connections (proxies or handles), so it can intercept and register how the client API interacts with the connection object.
* If the `DriverManager` is a *physical connection factory*, the `DataSource` interface is a *logical connection provider*. The simplest `DataSource` implementation could delegate connection acquisition requests to the underlying `DriverManager`, and the connection request workflow would look like this:
  1. data layer asks the `DataSource` for a connection
  2. `DataSource` uses the underlying driver to open a physical connection
  3. a physical connection is created, and a TCP socket is opened
  4. the `DataSource` under test does not wrap the physical connection, but lends it to the application layer
  5. the application executes statements using the acquired connection
  6. when the physical connection is no longer needed, it is closed along with the underlying TCP socket
* Opening and closing connections is an expensive operation, so reusing them has the following advantages:
  * avoid DB and driver overhead for establishing TCP connections
  * prevent destroying temporary memory buffers associated with each connection
  * reduce client-side JVM object garbage
* When using a connection pooling solution, the connection acquisition time is between two and four orders of magnitude smaller (the overall transaction response time decreases too).


#### Why is [pooling](https://www.baeldung.com/java-connection-pooling) so much faster?
* The connection pooling mechanism works like this:
  1. a connection is requested => the pool looks for unallocated connections
  2. if a free connection exists => hand it to the client
  3. if there are no free connections => the pool will try to grow to its max size
  4. if the pool is already at its max size, it will retry several times before giving up with a *connection acquisition failure exception*
  5. when the logical connection is closed, it is released to the pool without closing the underlying physical connection
* The connection pool does not return the physical connection to the client, but instead it offers a proxy. When a connection is in use, its state is changed to **allocated** to prevent two concurrent threads from using the same connection. The proxy intercepts the connection close method call, and it notifies the pool to change the connection state to **unallocated**. Apart from reducing connection acquisition time, the pooling mechanism can also limit the number of connections an application can use at once. The connection pool acts as a *bounded buffer* for the incoming connection requests. If there is a traffic spike, the connection pool will level it, instead of saturating all the available DB resources.
* Whenever the number of incoming requests surpasses available request handlers, there are two options to avoid system overloading:
  * discard the overflowing traffic (affecting availability)
  * queue requests and wait for busy resources to become available (increasing response time).

