# Notes on Java Persistence

* The fear of DB portability can lead to avoiding highly effective features just because they are not interchangeable across various DB systems. In reality, it is more common to end up with a sluggish DB layer than having to port an already running system to a new DB solution.
* [JPA](link) and [Hibernate](link) were never meant to substitute SQL, and [native queries](link) are unavoidable in any non-trivial enterprise application. While JPA makes it possible to abstract DML statements and common entity retrieval queries, when it comes to reading and processing data, nothing can beat native SQL.
* [JPQL (Java Persistence Querying Language)](link) abstracts the common SQL syntax that is supported by most relation DBs. Because of this, JPQL cannot take advantage of [Window Functions](link), [Common Table Expressions](link), [Derived tables](link) or [PIVOT](link).
* Just like [Criteria API](link) protects against [SQL injection attacks](link) when generating entity queries dynamically, [JOOQ](link) offers the same safety guarantee when building native SQL statements.

### Performance and scaling
* In application performance management, the two most important metrics are **response time** and **throughput**. The lower the response time, the more responsive an application becomes. **Response time** is, therefore, the **_measure of performance_**. Scaling is about maintaining low response times while increasing system load, so **throughput** is the **_measure of scalability_**.
* The transaction response time is measured as the sum of: 
  * the DB connection acquisition time
  * the time it takes to send all DB statements over the wire
  * the execution time for all incoming statements
  * the time it takes for sending the result sets back to the DB client
  * the time the transaction is idle due to application-level computations prior to releasing the DB connection
* Throughput can be calculated as the number of transactions executed within a given time interval:
  * X = transactionCount / time
* So, by lowering the transaction execution time, the system can accommodate more requests.
* Ideally, if the system were scaling linearly, adding more DB connections would yield a proportional throughput increase. Due to [contention](https://www.cockroachlabs.com/blog/what-is-DB-contention/) on DB resources and the cost of maintaining coherency across multiple concurrent DB sessions, the relative throughput gain follows a curve instead of a straight line.
* Every connection requires a [TCP](link) socket from the client (application) to the server (DB).
* A look into DB system internals reveals the tight dependency on CPU, Memory, and Disk resources. Because I/O operations are costly, the DB uses a buffer pool to map into memory the underlying data and index pages. Changes are first applied in memory and flushed to disk in batches to achieve better write performance.
* High-throughput DB applications experience contention on CPU, Memory, Disk, and Locks. When all the DB server resources are in use, adding more workload only increases contention, therefore lowering throughput.
* Lowering response time not only makes the application more responsive, but it can also increase throughput. However, response time alone is not sufficient in a highly concurrent environment. To maintain a fixed upper bound response time, the system capacity must increase relative to the incoming request throughput. Adding more resources can improve scalability up to a certain point, beyond which the capacity gain starts dropping.

#### Master-Slave replication
* For enterprise systems where the *read/write* ratio is high, a **Master-Slave** replication scheme is suitable for increasing availability. 
* The *Master* is the system of record and the only node accepting writes. All changes recorded by the *Master* node are replayed onto *Slaves* as well. A binary replication uses the *Master* node while a statement-based replication replays on the *Slave* machines the exact statements executed on Master. 
* Asynchronous replication is very common, especially when there are more *Slave* nodes to update.
* The *Slave* nodes are **eventual consistent** as they might lag behind the *Master*. In case the *Master* node crashes, a cluster-wide voting process must elect the new *Master* from the list of all available *Slaves*.
* The asynchronous replication topology is also referred as **_warm standby_** because the election process does not happen instantaneously.
* Most DB systems allow one synchronous *Slave* node, at the price of increasing transaction response time (the *Master* has to block waiting for the synchronous *Slave* node to acknowledge the update). In case of *Master* node failure, the automatic failover mechanism can promote the synchronous *Slave* node to become the new *Master*.
* Having one synchronous *Slave* allows the system to ensure data consistency in case of *Master* node failures since the synchronous *Slave* is an exact copy of the *Master*. The synchronous *Master-Slave* replication is also called a **_hot standby_** topology because the synchronous *Slave* is readily available for replacing the *Master* node.
* When only asynchronous *Slave* nodes are available, the newly elected *Slave* node might lag behind the failed *Master*, in which case **_consistency and durability_** are traded for **_lower latencies and higher throughput_**.
* Aside from eliminating the **single point of failure**, DB replication can also increase transaction throughput without affecting response time. In a *Master-Slave* topology, the *Slave* nodes can accept read-only transactions, therefore routing read traffic away from the *Master* node.
* The *Slave* nodes increase the available read-only connections and reduce *Master* node resource contention, which can also lower read-write transaction response time. If the *Master* node can no longer keep up with the ever-increasing read-write traffic, a Multi-Master replication might be a better alternative.


#### Multi-Master replication
* In a Multi-Master replication scheme, all nodes are equal and can accept both read-only and read-write transactions. Splitting the load among multiple nodes can only increase transaction throughput and reduce response time as well.
* However, because distributed systems are all about trade-offs, ensuring data consistency is challenging in a Multi-Master replication scheme because there is no longer a single source of truth. The same data can be modified concurrently on separate nodes, so there is a possibility of conflicting updates. The replication scheme can either avoid conflicts or it can detect them and apply an automatic conflict resolution algorithm.
* To avoid conflicts, the [two-phase commit](link) protocol can be used to enlist all participating nodes in one distributed transaction. This design allows all nodes to be in sync at all time, at the cost of increasing transaction response time (by slowing down write operations).
* If nodes are separated by a WAN (Wide Area Network), synchronization latencies may increase dramatically. If one node is no longer reachable, the synchronization will fail, and the transaction will roll back on all Masters.
* Although avoiding conflicts is better from a data consistency perspective, synchronous replication might incur high transaction response times. On the other hand, at the price of having to resolve update conflicts, asynchronous replication can provide better throughput. The asynchronous Multi-Master replication requires a conflict detection and an automatic conflict resolution algorithm. When a conflict is detected, the automatic resolution tries to merge the two conflicting branches, and, in case it fails, manual intervention is required.


#### Sharding
* When data size grows beyond the overall capacity of a replicated multi-node environment, splitting data becomes unavoidable. Sharding means distributing data across multiple nodes, so each instance contains only a subset of the overall data.
* Traditionally, relational DBs have offered *horizontal partitioning* to distribute data across multiple tables within the same DB server. As opposed to *horizontal partitioning*, **sharding** requires a distributed system topology so that data is spread across multiple machines.
* Each shard must be self-contained because a user transaction can only use data from within a single shard. Joining across shards is usually prohibited because the cost of distributed locking and the networking overhead would lead to long transaction response times.
* By reducing data size per node, indexes also require less space, and they can better fit into main memory. With less data to query, the transaction response time can also get shorter too.
* Each data center can serve a dedicated geographical region, so the load is balanced across geographical areas. Not all tables need to be partitioned across shards, smaller size ones being duplicated on each partition. To keep the shards in sync, an asynchronous replication mechanism can be employed.
* In the quest for increasing system capacity, sharding is usually a last resort strategy, employed after exhausting all other available options, such as:
  * optimizing the data layer to deliver lower transaction response times
  * scaling each replicated node to a cost-effective configuration
  * adding more replicated nodes until synchronization latencies start dropping below an acceptable threshold


### JDBC Connection Management
* The JDBC (Java DB Connectivity) API provides a common interface to communicate with the DB. All networking logic and DB-specific communication protocol are hidden behind the vendor-independent JDBC API. The `java.sql.Driver` is the entry point for interacting with the JDBC API, defining the implementation version details and providing access to a DB connection.
* JDBC defines 4 driver types, of which the *Type 4* (communication protocol implemented solely in Java) is the preferred alternative being easier to setup and debug.
* To communicate with the DB, a Java program first obtains a `java.sql.Connection`. Although the `java.sql.Driver` is the actual DB connection provider, it is more convenient to use the `java.sql.DriverManager` since it can also resolve the JDBC driver associated with the current DB connection URL.
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

### Batch Updates
* JDBC 2.0 introduced *batch updates* so that multiple DML statements can be grouped into a single DB request. Sending multiple statements in a single request reduces the number of DB roundtrips, therefore decreasing transaction response time. Even if the reference specification uses the term *updates*, any insert, update or delete statement can be batched, and JDBC supports batching for `java.sql.Statement` , `java.sql.PreparedStatement` and `java.sql.CallableStatement`.
* It is common to use a relatively small batch size, to reduce both the *client-side memory footprint* and to avoid congesting the server from suddenly processing a huge batch load.
* The **SQL Injection attack** exploits data access layers that do not use *bind parameters*. When the SQL statement is the result of *String concatenation*, an attacker could inject a malicious SQL routine that is sent to the DB along the current executing statement. SQL injection is usually done by ending the current statement with the **;** character and continuing it with a rogue SQL command, like modifying the DB structure (deleting a table or modifying authorization rights) or extracting sensitive information.
* Finding the right batch size is not easy as there is no mathematical equation to solve the appropriate batch size. Measuring the application performance gain in response to a certain batch size value remains the most reliable tuning option. A low batch size can reduce the transaction response time, and the performance gain does not grow linearly with the batch size. Although a larger batch value can save more DB *roundtrips*, the overall performance gain does not necessarily increase linearly. In fact, a very large batch size can hurt application performance if the transaction takes too long to be executed. As a rule of thumb, you should always measure the performance improvement for various batch sizes. In practice, a relatively low value (between 10 and 30) is usually a good choice.
* Processing too much data in a single transaction can degrade application performance, especially in a highly concurrent environment. Whether using [Two-Phase Locking](link) or [Multi-version Concurrency Control](link), writers always block other conflicting writers. Long-running transactions can affect both batch updates and bulk operations if the current transaction modifies a very large number of records. For this reason, it is more practical to break a large batch processing task into smaller manageable ones that can release locks in a timely fashion.

### Statement Caching
* Being a declarative language, SQL describes the *what* and not the *how*. Actual DB structures and algorithms used for fetching and preparing desired result sets are hidden away from the DB client, which only has to focus on properly defining SQL statements. This way, to deliver the most efficient *data access plan*, the DB can attempt various execution strategies.

#### Statement lifecycle
* The main DB modules responsible for processing a SQL statement are the **Parser**, the **Optimizer**, and the **Executor**.
* The *Parser* checks the SQL statement and ensures its validity. The statements are verified both *syntactically* and *semantically* (referenced tables and columns exist). During parsing, the SQL statement is transformed into a DB-internal representation, called the **syntax tree** (parse tree or query tree). If the SQL statement is a high-level representation, the syntax tree is the logical representation of the DB objects.
* For a given syntax tree, the DB must decide the most efficient data fetching algorithm. Data is retrieved by following an access path, and the Optimizer needs to evaluate multiple data traversing options like:
  * access method for each referencing table (*table scan* or *index scan*)
  * for index scans, it must decide which index is better suited for fetching this result set
  * for each joining relation (tables, views, Common Table Expression), it must choose the best-performing join type ([Nested Loop Joins](link), [Hash Joins](link), [Sort Merge Joins](link))
  * the joining order becomes important, especially for Nested Loops Joins
* The list of access path, chosen by the Optimizer, is assembled into an execution plan.
* The most common decision-making algorithm is the [Cost-Based Optimizer](link). Each access method translates to a physical DB operation and its associated cost in resources can be estimated. The DB stores various statistics like table size and [data cardinality](link) (how much the column values differ from one row to the other) to evaluate the cost of a given DB operation. The cost is calculated based on the number of CPU cycles and I/O operations required for executing a given plan. When finding an optimal execution plan, the Optimizer might evaluate multiple options and based on their overall cost it chooses the one requiring the least amount of time to execute.
* By now, it is clear that finding a proper execution plan is resource intensive and, for this purpose, some DB vendors offer execution plan caching. While caching can speed up statement execution, it also incurs some additional challenges (making sure the plan is still optimal across multiple executions). Each execution plan has a given memory footprint and most DB systems use a fixed-size cache (discarding the least used plans to make room for newer ones). [Data Definition Language](link) statements might corrupt execution plans making them obsolete, so the DB must use a separate process for validating the existing execution plans relevancy.
* From the Optimizer, the execution plan goes to the Executor where it is used to fetch the associated data and build the result set. The Executor makes use of the *Storage Engine* (for loading data according to the current execution plan) and the *Transaction Engine* (to enforce the current transaction data integrity guarantees). Having a reasonably large in-memory buffer allows the DB to reduce I/O contention, therefore reducing transaction response time. The consistency model also has an impact on the overall transaction performance since locks may be acquired to ensure data integrity, and the more locking, the less the chance for parallel execution.

##### An Example
* Assuming a *task* table has a *status* column with three distinct values: *TO_DO*, *DONE*, and *FAILED*. The table has 100_000 rows, of which 1_000 are *TO_DO* entries, 95_000 are *DONE*, and 4_000 are *FAILED* records. In DB terminology, the number of rows returned by a given predicate is called cardinality and, for the status column, the cardinality varies from 1000 to 95 000.
  * **C = {1000, 4000, 95000}**
* By dividing cardinality with the total number of rows, the predicate [selectivity](link) is obtained:
  * **S = C / N * 100 = {1%, 4%, 95%}**
* The lower the selectivity, the fewer rows are matched for a given bind value and the more selective the predicate gets. The Optimizer prefers sequential scans to index lookups for high selectivity percentages, to reduce the total number of disk-access round-trips (especially when data is scattered among multiple data blocks).


### Transactions
* A database system must allow concurrent access to the underlying data. However, shared data means that read and write operations must be synchronized to ensure that data integrity is not compromised. In a relational database, the mechanism for ensuring data integrity is implemented on top of transactions.
* A transaction is a collection of read and write operations that can either succeed or fail together, as a unit. All database statements must execute within a transactional context, even when the database client does not explicitly define its boundaries. 
* Knowing how database transactions work is very important for two main reasons:
  * effective data access (data integrity should not be compromised when aiming for high- performance)
  * efficient data access (reducing contention can minimize transaction response time which, in turn, increases throughput)

#### Atomicity
* Atomicity is the property of grouping multiple operations into an all-or-nothing unit of work, which can succeed only if all individual operations succeed. For this reason, the database must be able to roll back all actions associated with every executed statement.

##### Write-write conflicts
> Ideally, every transaction would have a completely isolated branch which could be easily discarded in case of a rollback. This scenario would be similar to how a Version Control System (e.g. git) implements branching. In case of conflicts, the Virtual Control System aborts the commit operation, and the client has to manually resolve the conflict. Unlike VCS tools, the relational database engine must manage conflicts without any human intervention. For this reason, the database prevents write-write conflict situations, and only one transac- tion can write a record at any given time.

#### Consistency
* A modifying transaction can be seen as a state transformation, moving the database from one valid state to another. The relational database schema ensures that all primary modifications (insert/update/delete statements), as well as secondary ones (issued by triggers), obey certain rules on the underlying data structures:
  * column types 
  * column length 
  * column nullability 
  * foreign key constraints 
  * unique key constraints 
  * custom check constraints
* Consistency is about validating the transaction state change so that all committed trans- actions leave the database in a proper state. If only one constraint gets violated, the entire transaction will be rolled back, and all modifications are going to be reverted.
* **Consistency as in CAP Theorem**: According to the CAP theorem a , when a distributed system encounters a network partition, the system must choose either Consistency (all changes are instantaneously applied to all nodes) or Availability (any node can accept a request), but not both. While in the definition of ACID, consistency is about obeying constraints, in the CAP theorem context, consistency refers to linearizability b , which is an isolation guarantee instead.

#### Isolation
* By offering multiple concurrent connections, the transaction throughput can increase, and the database system can accommodate more traffic. However, parallelization imposes addi- tional challenges as the database must interleave transactions in such a way that conflicts do not compromise data integrity. The execution order of all the currently running transaction operations is said to be serializable when its outcome is the same as if the underlying transactions were executed one after the other. The serializable execution is, therefore, the only transaction isolation level that does not compromise data integrity while allowing a certain degree of parallelization.

##### Concurrency control
* To manage data conflicts, several concurrency control mechanisms have been developed throughout the years. There are two strategies for handling data collisions:
  * Avoiding conflicts (e.g. two-phase locking) requires locking to control access to shared resources
  * Detecting conflicts (e.g. Multi-Version Concurrency Control) provides better concur- rency, at the price of relaxing serializability and possibly accepting various data anoma- lies

###### Two-phase locking
* Serializability could be obtained if all transactions used the 2PL protocol. It is important to understand the price of maintaining strict data integrity on the overall application scalability and transaction performance.
* Locking on lower levels (e.g. rows) can offer better concurrency as it reduces the likelihood of contention. Because each lock takes resources, holding multiple lower-level locks can add up, so the database might decide to substitute multiple lower-level locks into a single upper- level one. This process is called lock escalation, and it trades off concurrency for database resources.
* Each database system comes with its own lock hierarchy, but the most common types are:
  * shared (read) lock, preventing a record from being written while allowing reads
  * exclusive (write) lock, disallowing both read and write operations
* Locks alone are not sufficient for preventing conflicts. A concurrency control strategy must define how locks are being acquired and released because this also has an impact on transaction interleaving. For this purpose, the 2PL protocol defines a lock management strategy for ensuring serializ- ability. The 2PL protocol splits a transaction into two sections:
  * expanding phase (locks are acquired, and no lock is released)
  * shrinking phase (all locks are released, and no other lock is further acquired)
* In lock-based concurrency control, all transactions must follow the 2PL protocol, as other- wise serializability might be compromised, resulting in data anomalies.
* To provide recovery from failures, the transaction schedule (the sequence of all interleaved operations) must be strict. If a write operation, in a first transaction, happens-before a conflict occurring in a subsequent transaction, in order to achieve transaction strictness, the first transaction commit event must also happen before the conflict. Because operations are properly ordered, strictness can prevent cascading aborts (one trans- action rollback triggering a chain of other transaction aborts, to preserve data consistency). Releasing all locks only after the transaction has ended (either commit or rollback) is a requirement for having a strict schedule.

###### Multi-Version Concurrency Control
* Although locking can provide a serializable transaction schedule, the cost of lock contention can undermine both transaction response time and scalability. The response time can in- crease because transactions must wait for locks to be released, and long-running transactions can slow down the progress of other concurrent transactions as well. According to both Amdahl’s Law and the Universal Scalability Law, concurrency is also affected by contention. To address these shortcomings, the database vendors have opted for optimistic concurrency control mechanisms. If 2PL prevents conflicts, Multi-Version Concurrency Control (MVCC) uses a conflict detection strategy instead.
> The promise of MVCC is that readers do not block writers and writers do not block readers. The only source of contention comes from writers blocking other concurrent writers, which otherwise would compromise transaction rollback and atomicity.

##### Phenomena
* For reasonable transaction throughput values, it makes sense to imply transaction serializ- ability. As the incoming traffic grows, the price for strict data integrity becomes too high, and this is the primary reason for having multiple isolation levels. Relaxing serializability guarantees may generate data integrity anomalies, which are also referred as phenomena. The SQL-92 standard introduced three phenomena that can occur when moving away from a serializable transaction schedule:
  * dirty read 
  * non-repeatable read
  * phantom read
* In reality, there are other phenomena that can occur due to transaction interleaving:
  * dirty write 
  * read skew 
  * write skew 
  * lost update
> Choosing a certain isolation level is a trade-off between increasing concurrency and acknowl- edging the possible anomalies that might occur. Scalability is undermined by contention and coherency costs. The lower the isolation level, the less locking (or multi-version transaction abortions), and the more scalable the application gets. However, a lower isolation level allows more phenomena, and the data integrity responsibility is shifted from the database side to the application logic, which must ensure that it takes all measures to prevent or mitigate any such data anomaly.

###### Dirty write
* A dirty write happens when two concurrent transactions are allowed to modify the same row at the same time. As previously mentioned, all changes are applied to the actual database object structures, which means that the second transaction simply overwrites the first transaction pending change.
* If the two transactions commit, one transaction will silently overwrite the other transaction, causing a lost update. Another problem arises when the first transaction wants to roll back. The database engine would have to choose one of the following action paths:
  * It can restore the row to its previous version (as it was before the first transaction changed it), but then it overwrites the second transaction uncommitted change
  * It can acknowledge the existence of a newer version (issued by the second transaction), but then, if the second transaction has to roll back, its previous version will become the uncommitted change of the first transaction
* If the database engine did not prevent dirty writes, guaranteeing rollbacks would not be possible. Because atomicity cannot be implemented in the absence of reliable rollbacks, all database systems must, therefore, prevent dirty writes. Although the SQL standard does not mention this phenomenon, even the lowest isolation level (Read Uncommitted) can prevent it.

###### Dirty read
* As previously mentioned, all database changes are applied to the actual data structures (memory buffers, data blocks, indexes). A dirty read happens when a transaction is allowed to read the uncommitted changes of some other concurrent transaction. Taking a business decision on a value that has not been committed is risky because uncommitted changes might get rolled back.
* This anomaly is only permitted by the Read Uncommitted isolation level, and, because of the serious impact on data integrity, most database systems offer a higher default isolation level. To prevent dirty reads, the database engine must hide uncommitted changes from all other concurrent transactions. Each transaction is allowed to see its own changes because otherwise the read-your-own-writes consistency guarantee is compromised.
> Read Uncommitted is rarely needed (non-strict reporting queries where dirty reads are acceptable), so Read Committed is usually the lowest practical isolation level.

###### Non-repeatable read
* If one transaction reads a database row without applying a shared lock on the newly fetched record, then a concurrent transaction might change this row before the first transaction has ended.
* This phenomenon is problematic when the current transaction makes a business decision based on the first value of the given database row (a client might order a product based on a stock quantity value that is no longer a positive integer). Most database systems have moved to a Multi-Version Concurrency Control model, and shared locks are no longer mandatory for preventing non-repeatable reads. By verifying the current row version, a transaction can be aborted if a previously fetched record has changed in the meanwhile. Repeatable Read and Serializable prevent this anomaly by default. With Read Committed, it is possible to avoid non-repeatable (fuzzy) reads if the shared locks are acquired explicitly (e.g. SELECT FOR SHARE ). Some ORM frameworks (e.g. JPA/Hibernate) offer application-level repeatable reads. The first snapshot of any retrieved entity is cached in the currently running Persistence Context. Any successive query returning the same database row is going to use the very same object that was previously cached. This way, the fuzzy reads may be prevented even in Read Committed isolation level.

###### Phantom read
* If a transaction makes a business decision based on a set of rows satisfying a given predicate, without range locks, a concurrent transaction might insert a record matching that particular predicate.
* The SQL standard says that Phantom Read occurs if two consecutive query executions render different results because a concurrent transaction has modified the range of records in between the two calls. Although providing consistent reads is a mandatory requirement for serializability, that is not sufficient. For instance, one buyer might purchase a product without being aware of a better offer that was added right after the user has finished fetching the offer list.

###### Lost update
* This phenomenon happens when a transaction reads a row while another transaction modifies it prior to the first transaction to finish.
* This anomaly can have serious consequences on data integrity (a buyer might purchase a product without knowing the price has just changed), especially because it affects Read Committed, the default isolation level in many database systems. Traditionally, Repeatable Read protected against lost updates since the shared locks could prevent a concurrent transaction from modifying an already fetched record. With MVCC, the second transaction is allowed to make the change, while the first transaction is aborted when the database engine detects the row version mismatch (during the first transaction commit). Most ORM tools, such as Hibernate, offer application-level optimistic locking, which auto- matically integrates the row version whenever a record modification is issued. On a row version mismatch, the update count is going to be zero, so the application can roll back the current transaction, as the current data snapshot is stale.

##### Isolation levels
* As previously stated, Serializable is the only isolation level to provide a truly ACID transaction interleaving. However, serializability comes at a price as locking introduces contention, which, in turn, limits concurrency and scalability. Even in multi-version concurrency models, serializability may require aborting too many transactions that are affected by phenomena. For this purpose, the SQL-92 version introduced multiple isolation levels, and the database client has the option of balancing concurrency against data correctness. Each isolation level is defined in terms of the minimum number of phenomena that it must prevent, and so the SQL standard introduces the following transaction isolation levels:

| Isolation Level  | Dirty Read | Non-repeatable Read | Phantom Read |
|------------------|------------|---------------------|--------------|
 | Read Uncommitted | Yes        | Yes                 | Yes          |
 | Read Committed   | No         | Yes                 | Yes          |
 | Repeatable Read  | No         | No                  | Yes          |
 | Serializable     | No         | No                  | No           |

* Even if ACID properties imply a serializable schedule, most relational database systems use a lower default isolation level instead:
  * Read Committed (Oracle, SQL Server, PostgreSQL)
  * Repeatable Read (MySQL)

###### Read Uncommitted (TODO)

###### Read Committed
* Read Committed is one of the most common isolation levels, and it behaves consistently across multiple relational database systems or various concurrency control models. Many database systems choose Read Committed as the default isolation level because it delivers the best performance while preventing fatal anomalies such as dirty writes and dirty reads. However, performance has its price as Read Committed permits many anomalies that might lead to data corruption.

###### Repeatable Read (TODO)

###### Serializable
* Serializable is supposed to provide a transaction schedule, whose outcome, even in spite of statement interleaving, is equivalent to a serial execution. Even if the concurrency control mechanism is lock-based or it manages multiple record versions, it must prevent all phenomena to ensure serializable transactions. Preventing all phenomena mentioned by the SQL standard (dirty reads, non-repeatable reads and phantom reads) is not enough, and Serializable must protect against lost update, read skew and write skew as well. In practice, the concurrency control implementation details leak, and not all relational database systems provide a truly Serializable isolation level (some data integrity anomalies might still occur).

##### Durability
* Durability ensures that all committed transaction changes become permanent. Durability allows system recoverability, and, to some extent, it is similar to the rolling back mechanism.

#### Read-only transactions
* The JDBC Connection defines the setReadOnly(boolean readOnly) 7 method which can be used to hint the driver to apply some database optimizations for the upcoming read-only transac- tions. This method should not be called in the middle of a transaction because the database system cannot turn a read-write transaction into a read-only one (a transaction must start as read-only from the very beginning).

#### Distributed transactions
* The difference between local and global transactions is that the former uses a single resource manager, while the latter operates on multiple heterogeneous resource managers. The ACID guarantees are still enforced on each individual resource, but a global transaction manager is mandatory to orchestrate the distributed transaction outcome. All transactional resource adapters are registered by the global transaction manager, which decides when a resource is allowed to commit or rollback. The Java EE managed resources become accessible through JNDI (Java Naming and Directory Interface) or CDI (Contexts and Dependency Injection). Spring provides a transaction management abstraction layer which can be configured to either use local transactions (JDBC or RESOURCE_LOCAL 8 JPA) or global transactions through a stand-alone JTA transaction manager. The dependency injection mechanism auto- wires managed resources into Spring beans.
  
##### Two-phase commit
* JTA makes use of the two-phase commit (2PC) protocol to coordinate the atomic resource commitment in two steps: a prepare and a commit phase. In the former phase, a resource manager takes all the necessary actions to prepare the trans- action for the upcoming commit. Only if all resource managers successfully acknowledge the preparation step, the transaction manager proceeds with the commit phase. If one resource does not acknowledge the prepare phase, the transaction manager will proceed to roll back all current participants. If all resource managers acknowledge the commit phase, the global transaction will end successfully. If one resource fails to commit (or times out), the transaction manager will have to retry this operation in a background thread until it either succeeds or reports the incident for manual intervention.

##### Declarative transactions
* Transaction boundaries are usually associated with a Service layer, which uses one or more DAO to fulfill the business logic. The transaction propagates from one component to the other within the service-layer transaction boundaries.

##### Propagation levels
| Propagation   | Description                                                                                                                                                        |
|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
 | REQUIRED      | This is the default propagation strategy, and it only starts a transaction unless the current thread is not already associated with a transaction context          |
 | REQUIRES_NEW  | Any currently running transaction context is suspended and replaced by a new transaction                                                                           |
 | SUPPORTS      | If the current thread already runs inside a transaction, this method will use it. Otherwise, it executes outside of a transaction context                          |
 | NOT_SUPPORTED | Any currently running transaction context is suspended, and the current method is run outside of a transaction context                                             |
 | MANDATORY     | The current method runs only if the current thread is already associated with a transaction context                                                                |
 | NESTED        | The current method is executed within a nested transaction if the current thread is already associated with a transaction. Otherwise, a new transaction is started |
 | NEVER         | The current method must always run outside of a transaction context, and, if the current thread is associated with a transaction, an exception will be thrown      |

##### Pessimistic locking
* Most database systems already offer the possibility of manually requesting shared or exclusive locks. This concurrency control is said to be pessimistic because it assumes that conflicts are bound to happen, and so they must be prevented accordingly. As locks can be released in a timely fashion, exclusive locking is appropriate during the last database transaction of a given long conversation. This way, the application can guarantee that, once locks are acquired, no other transaction can interfere with the currently locked resources. Acquiring locks on critical records can prevent non-repeatable reads, lost updates, as well as read and write skew phenomena.

##### Optimistic locking
* The optimistic locking concurrency algorithm looks like this:
  * When a client reads a particular row, its version comes along with the other fields
  * Upon updating a row, the client filters the current record by the version it has previously loaded
  * If the statement update count is zero, the version was incremented in the meanwhile, and the current transaction now operates on a stale record version


### Why JPA and Hibernate matter
* Although JDBC does a very good job of exposing a common API that hides the database vendor-specific communication protocol, it suffers from the following shortcomings:
  * The API is undoubtedly verbose, even for trivial tasks
  * Batching is not transparent from the data access layer perspective, requiring a specific API than its non-batched statement counterpart
  * Lack of built-in support for explicit locking and optimistic concurrency control
  * For local transactions, the data access is tangled with transaction management semantics
  * Fetching joined relations requires additional processing to transform the ResultSet into Domain Models or DTO (Data Transfer Object) graphs

#### The impedance mismatch
* When a relational database is manipulated through an object-oriented program, the two different data representations start conflicting. In a relational database, data is stored in tables, and the relational algebra defines how data associations are formed. On the other hand, an object-oriented programming (OOP) language allows objects to have both state and behavior, and bidirectional associations are permitted.
* Because the underlying data resides in a relational database, the Domain Model must be adapted to the database schema and the SQL-driven communication protocol. The ORM design pattern helps to bridge these two different data representations and close the tech- nological gap between them. Every database row is associated with a Domain Model object (Entity in JPA terminology), and so the ORM tool can translate the entity state transitions into DML statements. From an application development point of view, this is very convenient since it is much easier to manipulate Domain Model relationships rather than visualizing the business logic through its underlying SQL statements.

#### JPA vs Hibernate
* JPA is only a specification. It describes the interfaces that the client operates with and the standard object-relational mapping metadata (Java annotations or XML descriptors). Beyond the API definition, JPA also explains (although not exhaustively) how these specifications are ought to be implemented by the JPA providers.
* Hibernate was already a full-featured Java ORM implementation by the time the JPA specifica- tion was released for the first time. Although it implements the JPA specification, Hibernate retains its native API for both backward compatibility and to accommodate non-standard features.
> If JPA is the interface, Hibernate is one implementation and implementation details always matter from a performance perspective.

#### Entity state transitions
* JPA shifts the developer mindset from SQL statements to entity state transitions. An entity can be in one of the following states:

| State                    | Description                                                                                                                                                                                                                                               |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
 | **New** (Transient)      | A newly created entity which is not mapped to any database row is considered to be in the New or Transient state. Once it becomes managed, the Persistence Context issues an insert statement at flush time                                               |
 | **Managed** (Persistent) | A Persistent entity is associated with a database row, and it is being managed by the currently running Persistence Context. State changes are detected by the dirty checking mechanism and propagated to the database as update statements at flush time |
 | **Detached**             | Once the currently running Persistence Context is closed, all the previously managed entities become detached. Successive changes are no longer tracked, and no automatic database synchronization is going to happen                                     |
 | **Removed**              | A removed entity is only scheduled for deletion, and the actual database delete statement is executed during Persistence Context flushing                                                                                                                 |

* The Persistence Context captures entity state changes, and, during flushing, it translates them into SQL statements. The JPA EntityManager 4 and the Hibernate Session 5 (which includes additional methods for moving an entity from one state to the other) interfaces are gateways towards the underlying Persistence Context, and they define all the entity state transition operations.

#### Write-based optimizations
> **SQL injection prevention** By managing the SQL statement generation, the JPA tool can assist in minimizing the risk of SQL injection attacks. The less the chance of manipulating SQL String statements, the safer the application can get. The risk is not completely eliminated because the application developer can still recur to concatenating SQL or JPQL fragments, so rigor is advised. Hibernate uses PreparedStatement(s) exclusively, so not only it protects against SQL injection, but the data access layer can better take advantage of server-side and client- side statement caching as well.
> **Auto-generated DML statements** The enterprise system database schema evolves with time, and the data access layer must mirror all these modifications as well. Because the JPA provider auto-generates insert and update statements, the data access layer can easily accommodate database table structure modifications. By updating the entity model schema, Hibernate can automatically adjust the modifying statements accordingly. This applies to changing database column types as well. If the database schema needs to migrate a postal code from an INT database type to a VARCHAR(6) , the data access layer will need only to change the associated Domain Model attribute type from an Integer to a String , and all statements are going to be automatically updated. Hibernate defines a highly customizable JDBC-to-database type mapping system, and the application developer can override a default type association, or even add support for new database types (that are not currently supported by Hibernate). The entity fetching process is automatically managed by the JPA implementation, which auto- generates the select statements of the associated database tables. This way, JPA can free the application developer from maintaining entity selection queries as well. Hibernate allows customizing all the CRUD statements, in which case the application devel- oper is responsible for maintaining the associated DML statements. Although it takes care of the entity selection process, most enterprise systems need to take advantage of the underlying database querying capabilities. For this reason, whenever the database schema changes, all the native SQL queries need to be updated manually (according to their associated business logic requirements).

##### Write-behind cache
* The Persistence Context acts as a transactional write-behind cache, deferring entity state flushing up until the last possible moment. Because every modifying DML statement requires locking (to prevent dirty writes), the write behind cache can reduce the database lock acquisition interval, therefore increasing concurrency. However, caches introduce consistency challenges, and the Persistence Context requires a flush prior to executing any JPQL or native SQL query, as otherwise it might break the read-your-own-write consistency guarantee.

#### Read-based optimizations
* Following the SQL standard, the JDBC ResultSet is a tabular representation of the underlying fetched data. The Domain Model being constructed as an entity graph, the data access layer must transform the flat ResultSet into a hierarchical structure. Although the goal of the ORM tool is to reduce the gap between the object-oriented Domain Model and its relational counterpart, it is very important to remember that the source of data is not an in-memory repository, and the fetching behavior influences the overall data access efficiency.
> The database cannot be abstracted out of this context, and pretending that entities can be manipulated just like any other plain objects is very detrimental to application performance. When it comes to reading data, the impedance mismatch becomes even more apparent, and, for performance reasons, it is mandatory to keep in mind the SQL statements associated with every fetching operation.
> **Prefer projections for read-only views**: Although it is very convenient to fetch entities along with all their associated relationships, it is better to take into consideration the performance impact as well. As previously explained, fetching too much data is not suitable because it increases the transaction response time. In reality, not all use cases require loading entities, and not all read operations need to be served by the same fetching mechanism. Sometimes a custom projection (selecting only a few columns from an entity) is much more suitable, and the data access logic can even take advantage of database-specific SQL constructs that might not be supported by the JPA query abstraction. As a rule of thumb, fetching entities is suitable when the logical transaction requires modifying them, even if that only happens in a successive web request. With this is mind, it is much easier to reason on which fetching mechanism to employ for a given business logic use case.
> **The second-level cache**: If the Persistence Context acts as a transactional write-behind cache, its lifetime will be bound to that of a logical transaction. For this reason, the Persistence Context is also known as the first-level cache, and so it cannot be shared by multiple concurrent transactions. On the other hand, the second-level cache is associated with an EntityManagerFactory , and all Persistence Contexts have access to it. The second-level cache can store entities as well as entity associations (one-to-many and many-to-many relationships) and even entity query results. Because JPA does not make it mandatory, each provider takes a different approach to caching (as opposed to EclipseLink, by default, Hibernate disables the second-level cache). Most often, caching is a trade-off between consistency and performance. Because the cache becomes another source of truth, inconsistencies might occur, and they can be prevented only when all database modifications happen through a single EntityManagerFactory or a synchronized distributed caching solution. In reality, this is not practical since the application might be clustered on multiple nodes (each one with its own EntityManagerFactory ) and the database might be accessed by multiple applications. Although the second-level cache can mitigate the entity fetching performance issues, it requires a distributed caching implementation, which might not elude the network- ing penalties anyway.

