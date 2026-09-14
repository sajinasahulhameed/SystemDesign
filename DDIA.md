#  System Design Master Guide

> **Purpose:** My understanding system-design thinking.
>
> **Source boundary:** Sections marked **DDIA --- Source** summarize and
> explain the idea. Sections marked **System
> Design --- Added** are deliberately additional teaching material:
> design heuristics, interview structure, diagrams, and practical
> connections that are not presented as quotations from the book.
>
> **Important:** This is a learning guide, not a replacement for the
> book. It intentionally compresses the 613-page source into a reusable
> reference.

------------------------------------------------------------------------

## How to use this guide

Use the following loop for every chapter:

**Learn → Explain → Design → Break → Fix → Repeat**

For each topic, ask:

1.  What problem is this solving?
2.  What assumptions does the solution make?
3.  What happens when a machine, network, process, or person fails?
4.  What is the bottleneck?
5.  What consistency or correctness guarantee is required?
6.  What happens as traffic/data/users increase?
7.  What does the operator have to do at 3 a.m.?
8.  What trade-off did we buy?

### Core mental model

Most system-design discussions can be reduced to a chain:

``` text
Requirements
    ↓
Workload
    ↓
Data model
    ↓
Storage / indexes
    ↓
Caching
    ↓
Partitioning
    ↓
Replication
    ↓
Async processing
    ↓
Consistency / transactions
    ↓
Failure handling
    ↓
Observability + operations
```

Do not memorize architectures as recipes. Learn why each box exists.

------------------------------------------------------------------------

# Part I --- Foundations of Data Systems

## Chapter 1 --- Reliable, Scalable, and Maintainable Applications

### DDIA --- Source

The book begins by framing many modern applications as **data-intensive
rather than compute-intensive**. The difficult part is often the amount
of data, the complexity of the data, or how quickly it changes.

Common building blocks include:

-   databases --- store data for later retrieval
-   caches --- retain expensive-to-compute results for faster reads
-   search indexes --- support keyword/filter queries
-   stream processing --- asynchronously process messages/events
-   batch processing --- periodically process accumulated data

The important point is that these components have different
characteristics and access patterns. There is no universal data system
that is best for every workload.

### Reliability

A system is reliable when it continues to work correctly even when
faults occur.

The book distinguishes several sources of failure:

-   **hardware faults** --- disks, machines, power, networks
-   **software errors** --- bugs, unexpected interactions, resource
    exhaustion
-   **human errors** --- incorrect configuration, deployment mistakes,
    operational mistakes

A reliable design therefore does more than hope individual components
never fail. It anticipates faults and limits their impact.

### Scalability

Scalability is not simply "can it handle a lot of users?"

You must describe:

-   **load** --- requests/sec, reads/sec, writes/sec, data volume,
    fan-out, etc.
-   **performance** --- response time, throughput, and how these change
    as load increases

A useful design conversation starts by identifying the load parameters
that actually matter.

### Maintainability

The book presents three major concerns:

-   **Operability:** make it easy for operations teams to keep the
    system healthy.
-   **Simplicity:** manage complexity so engineers can understand the
    system.
-   **Evolvability:** make future changes easier.

A system that works today but becomes impossible to understand or modify
is not a successful long-term design.

### The key lesson

There is no single "scaling trick." Reliability, scalability, and
maintainability emerge from multiple design decisions that interact.

------------------------------------------------------------------------

## System Design --- Added: the requirement-first framework

Before choosing Kafka, Redis, PostgreSQL, Cassandra, Kubernetes, or
anything else, write the requirements.

``` mermaid
flowchart TD
    A[Product requirements] --> B[Functional requirements]
    A --> C[Non-functional requirements]
    B --> D[APIs + data model]
    C --> E[Availability]
    C --> F[Latency]
    C --> G[Throughput]
    C --> H[Durability]
    C --> I[Consistency]
    D --> J[Architecture]
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J
```

### Practical requirement template

  Dimension      Question
  -------------- --------------------------------
  Users          How many active users?
  Reads          Reads/sec?
  Writes         Writes/sec?
  Data           Current and yearly growth?
  Latency        p50/p95/p99 target?
  Availability   What downtime is acceptable?
  Durability     Can data ever be lost?
  Consistency    How stale may reads be?
  Geography      One region or many?
  Cost           What is the budget constraint?

### Expert habit

When someone says "make it scalable," respond internally with:

> **Scalable with respect to which dimension?**

A system can scale writes while failing to scale reads, or scale request
count while failing to scale storage, fan-out, or operational
complexity.

------------------------------------------------------------------------

# Chapter 2 --- Data Models and Query Languages

## DDIA --- Source

The chapter compares major ways of modeling data.

### Relational model

The relational model represents data using tables, rows, columns,
relationships, and SQL.

Its strengths include:

-   powerful joins
-   many-to-many relationships
-   mature query languages
-   explicit structure
-   strong transactional tooling in many systems

### Document model

Document databases represent related data as self-contained documents.

The model can work well when:

-   data naturally forms document-shaped aggregates
-   relationships between documents are relatively rare
-   nested data is frequently accessed together

But embedding and duplication introduce their own trade-offs.

### NoSQL

The book describes NoSQL not as one single model but as a family of
approaches. Two major directions discussed are:

1.  document databases
2.  graph databases

The broader lesson is that different data models fit different access
patterns.

### Relational vs document thinking

A relational model often encourages normalization and relationships
between entities.

A document model often encourages putting data that is accessed together
into the same aggregate.

Neither is universally superior.

### Query languages

The chapter discusses SQL, MapReduce, MongoDB's aggregation pipeline,
Cypher, SPARQL, and Datalog.

The key principle is that query languages reflect the underlying data
model. A language that makes one class of query elegant can make another
awkward.

### Graph data

Graphs represent:

-   **vertices/nodes** --- entities
-   **edges** --- relationships

They are useful when relationships are central to the problem, such as
social networks, dependency graphs, or highly connected domain data.

------------------------------------------------------------------------

## System Design --- Added: model from access patterns

Start with queries, not database branding.

Example:

``` text
Need:
1. Get user profile by ID
2. Get user's latest 50 posts
3. Get posts from followed users
4. Search posts by keyword
```

These are four different access patterns.

``` mermaid
flowchart LR
    A[User Profile] --> B[Primary DB]
    C[Recent Posts] --> B
    D[Feed Generation] --> E[Feed Service]
    E --> B
    F[Keyword Search] --> G[Search Index]
```

The database decision should follow the access patterns.

### Data-model checklist

Ask:

-   What is the natural aggregate?
-   What relationships exist?
-   Which reads must be fast?
-   Which writes must be atomic?
-   Is the workload transactional or analytical?
-   Is the data mostly key-based lookup, range query, full-text search,
    or graph traversal?
-   How much duplication is acceptable?

### Anti-pattern

> "Use NoSQL because the system needs to scale."

Scaling is not a data model. Determine the access pattern and scaling
constraint first.

------------------------------------------------------------------------

# Chapter 3 --- Storage and Retrieval

## DDIA --- Source

This chapter moves beneath database APIs into storage-engine internals.

A major lesson is:

> Different storage engines are optimized for different workloads.

### Hash indexes

A hash index maps a key to a location/value and can provide efficient
key-based lookup.

The trade-off is that hash-based indexing does not naturally support
ordered range queries.

### SSTables and LSM-trees

Log-structured approaches write data sequentially and later compact
sorted files.

The basic idea:

``` text
Writes
  ↓
In-memory structure
  ↓
Flush
  ↓
Sorted immutable file
  ↓
Compaction
  ↓
Larger sorted files
```

This can turn random writes into sequential writes and provide strong
write throughput.

### B-trees

B-trees use fixed-size pages and update data in place.

They are widely used in relational databases and many other systems.

### LSM-tree vs B-tree

Think in terms of workload rather than ideology.

  -----------------------------------------------------------------------
  Concern                 LSM-oriented design     B-tree-oriented design
  ----------------------- ----------------------- -----------------------
  Write pattern           sequential/log          in-place page updates
                          structured              

  Compaction              important               not the same central
                                                  mechanism

  Read amplification      can arise from multiple tree traversal
                          levels                  

  Write amplification     can arise from          can arise from page
                          compaction              updates

  Range queries           supported               naturally strong

  Best choice             depends on workload     depends on workload
  -----------------------------------------------------------------------

### OLTP vs analytics

OLTP workloads tend to perform many small reads/writes.

Analytical workloads often scan large volumes of data and aggregate it.

The book explains why column-oriented storage is valuable for analytical
queries: when a query needs only a few columns from huge numbers of
rows, storing values by column can avoid reading irrelevant data.

### Materialized views and data cubes

A materialized view stores precomputed query results.

That can make repeated reads fast, but the derived result must be
updated when source data changes.

------------------------------------------------------------------------

## System Design --- Added: choose storage by access pattern

``` mermaid
flowchart TD
    A[What query dominates?] --> B{Key lookup?}
    B -->|Yes| C[Key-value / indexed DB]
    B -->|No| D{Range query?}
    D -->|Yes| E[Ordered index / B-tree-like structure]
    D -->|No| F{Analytics scan?}
    F -->|Yes| G[Columnar / analytical store]
    F -->|No| H{Text search?}
    H -->|Yes| I[Search index]
    H -->|No| J{Relationship traversal?}
    J -->|Yes| K[Graph-oriented model]
```

### Useful expert vocabulary

When performance is poor, think about:

-   read amplification
-   write amplification
-   space amplification
-   cache hit rate
-   sequential vs random I/O
-   compaction
-   indexing cost
-   working-set size

You do not need to memorize every storage engine. You need to understand
the physical consequences of your workload.

------------------------------------------------------------------------

# Chapter 4 --- Encoding and Evolution

## DDIA --- Source

Data is often encoded when it moves between processes or is stored.

The chapter examines:

-   language-specific serialization
-   JSON
-   XML
-   binary formats
-   Thrift
-   Protocol Buffers
-   Avro
-   schemas
-   database dataflow
-   service-to-service dataflow
-   message-passing dataflow

### Evolution matters

Software changes over time.

Therefore, data formats must survive situations where:

-   old code reads new data
-   new code reads old data
-   producers and consumers are upgraded independently

This is why compatibility is central.

### Schema evolution

A schema can be explicit and enforced, or implicit and handled by
application logic. The book emphasizes that even systems without an
enforced schema generally have an assumed structure.

### Modes of dataflow

The book discusses data flowing:

1.  through databases
2.  through services via APIs/RPC
3.  through asynchronous message passing

The architectural question is not merely "how do I serialize this
object?" It is:

> **Who writes the data, who reads it, and how independently can those
> components evolve?**

------------------------------------------------------------------------

## System Design --- Added: compatibility as a first-class requirement

A useful migration pattern:

``` mermaid
sequenceDiagram
    participant Old as Old Consumer
    participant New as New Producer
    participant Bus as Data/API Boundary
    Old->>Bus: Read compatible format
    New->>Bus: Write new fields
    Bus-->>Old: Old fields still usable
    Bus-->>Old: Unknown fields ignored
```

### Safe evolution checklist

For every API/event/schema change ask:

-   Can old readers understand it?
-   Can new readers understand old records?
-   Can producers and consumers deploy independently?
-   What happens to data already persisted?
-   Can the change be rolled back?
-   What happens during a mixed-version deployment?

### Design lesson

**Backward compatibility is an architectural feature, not just a
serialization detail.**

------------------------------------------------------------------------

# Part II --- Distributed Data

# Chapter 5 --- Replication

## DDIA --- Source

Replication means keeping copies of data on multiple machines.

Reasons include:

-   reducing latency by placing data nearer users
-   increasing availability
-   increasing read throughput

But replication introduces the problem of keeping copies consistent.

### Single-leader replication

One node accepts writes; followers replicate changes.

``` mermaid
flowchart LR
    A[Application] --> L[Leader]
    L --> F1[Follower 1]
    L --> F2[Follower 2]
    A --> R1[Read path]
    R1 --> F1
```

### Synchronous vs asynchronous replication

Synchronous replication waits for confirmation from another replica
before completing the write.

Asynchronous replication allows the leader to acknowledge before
followers have caught up.

The trade-off is fundamentally about latency, availability, and how much
replication lag you tolerate.

### Node failures

The book discusses:

-   setting up new followers
-   follower outages
-   leader outages
-   replication logs
-   failover

### Replication lag

With asynchronous replication, a follower can temporarily be behind.

This produces user-visible anomalies such as:

-   **reading your own writes** failure
-   **monotonic reads** failure
-   **consistent prefix reads** failure

### Multi-leader replication

Multiple nodes accept writes.

This can be useful for:

-   multi-datacenter operation
-   collaborative/offline applications
-   geographically distributed deployments

But concurrent writes can conflict.

### Leaderless replication

Clients may write to multiple replicas and use quorum-like approaches
for reads and writes.

The chapter also discusses sloppy quorums, hinted handoff, and detecting
concurrent writes.

------------------------------------------------------------------------

## System Design --- Added: replication decision tree

``` mermaid
flowchart TD
    A[Need replication?] --> B{Mostly reads?}
    B -->|Yes| C[Leader + read replicas may fit]
    B -->|No| D{Writes in multiple locations?}
    D -->|Yes| E[Consider multi-leader / leaderless]
    D -->|No| F[Single leader may be simpler]
    C --> G{Can reads be stale?}
    G -->|Yes| H[Async replicas]
    G -->|No| I[Stronger coordination]
```

### Critical interview distinction

**Replication is not partitioning.**

-   Replication = copies of the same data.
-   Partitioning = splitting data across nodes.

You often use both.

``` text
                Dataset
                   |
          +--------+--------+
          |                 |
      Partition A       Partition B
       /      \           /      \
   Replica  Replica   Replica  Replica
```

------------------------------------------------------------------------

# Chapter 6 --- Partitioning

## DDIA --- Source

Partitioning (often called sharding) divides a dataset across nodes so
each node stores only part of the data.

The goal is to distribute:

-   storage
-   reads
-   writes
-   computation

### Key-range partitioning

Keys are divided into ranges.

Advantages:

-   range queries can be efficient
-   ordered data is preserved

Risk:

-   hot spots when traffic is concentrated in a small range

### Hash partitioning

A hash of the key determines the partition.

This tends to distribute keys more evenly.

Trade-off:

-   range queries become less naturally localized

### Skew and hot spots

A partitioning strategy can be mathematically balanced but operationally
unbalanced.

Example:

``` text
Popular key = celebrity_user
       ↓
Partition 7 receives huge traffic
       ↓
Partition 7 becomes bottleneck
```

### Secondary indexes

The chapter discusses two broad approaches:

-   partition secondary indexes by document
-   partition secondary indexes by term

Each produces different query and write trade-offs.

### Rebalancing

As nodes are added or removed, partitions must move.

The book emphasizes that rebalancing is a critical operational concern,
not an afterthought.

### Request routing

A distributed system must determine which node should receive a request.

------------------------------------------------------------------------

## System Design --- Added: partitioning strategy

``` mermaid
flowchart TD
    A[Choose partition key] --> B{Need range queries?}
    B -->|Yes| C[Key range]
    B -->|No| D[Hash key]
    C --> E[Watch for hot ranges]
    D --> F[Watch for hot keys]
    E --> G[Rebalancing]
    F --> G
    G --> H[Routing]
```

### Good partition key properties

A useful partition key generally has:

-   high enough cardinality
-   reasonably even distribution
-   predictable query routing
-   acceptable locality for common queries

### The hardest partitioning question

> **What query will become expensive after the dataset is split?**

A partition key can make writes easy and reads terrible, or vice versa.

------------------------------------------------------------------------

# Chapter 7 --- Transactions

## DDIA --- Source

Transactions are a mechanism for grouping operations so the system can
provide useful correctness guarantees despite failures and concurrency.

### ACID

The chapter explains the four letters:

-   **Atomicity:** operations in a transaction behave as one unit.
-   **Consistency:** application-defined invariants are preserved.
-   **Isolation:** concurrent transactions do not improperly interfere.
-   **Durability:** committed data survives failures.

A crucial nuance: **consistency is not simply a database property**. The
meaning depends on application invariants and guarantees.

### Weak isolation

The chapter examines:

-   read committed
-   snapshot isolation / repeatable read
-   lost updates
-   write skew
-   phantoms

### Serializability

Serializable execution aims to provide behavior equivalent to some
serial ordering of transactions.

Approaches discussed include:

-   actual serial execution
-   two-phase locking
-   serializable snapshot isolation

### Why transactions matter

Transactions are not primarily about syntax like `BEGIN` and `COMMIT`.

They are about defining what must remain true when:

-   requests race
-   processes crash
-   clients retry
-   operations partially succeed
-   multiple records change together

------------------------------------------------------------------------

## System Design --- Added: transaction boundary design

Suppose transferring money changes two accounts.

``` mermaid
flowchart LR
    A[Account A -100] --> T[Transaction]
    T --> B[Account B +100]
    T --> C[Invariant: total money preserved]
```

Ask:

1.  Must both changes happen atomically?
2.  Can the records live on different partitions?
3.  What happens if the client retries?
4.  What happens after timeout but before the client learns whether
    commit succeeded?

### Idempotency

A powerful distributed-systems technique:

``` text
request_id = 12345

First request:
  process → commit result

Retry:
  request_id already seen
  → return previous result
```

This is especially important for payment APIs, job submission, and other
operations where clients may retry after timeouts.

### Expert habit

Do not ask only:

> "Do we need a transaction?"

Ask:

> **Which invariant must hold, and across which pieces of state?**

------------------------------------------------------------------------

# Chapter 8 --- The Trouble with Distributed Systems

## DDIA --- Source

Distributed systems are difficult because components can fail
independently and communication is imperfect.

### Partial failure

In a single process, failure may be obvious.

In a distributed system:

``` text
Client ---- request ----> Server
Client <--- ??? --------- Server
```

A timeout does not necessarily tell you whether:

-   the request never arrived
-   the server is slow
-   the server completed the operation
-   the response was lost

### Unreliable networks

Networks can:

-   delay messages
-   drop messages
-   duplicate messages
-   reorder messages
-   disconnect
-   partially fail

### Time

Distributed systems also have difficult relationships with clocks.

The chapter distinguishes:

-   monotonic clocks
-   time-of-day clocks
-   clock synchronization
-   clock accuracy

It discusses why relying on synchronized clocks can be dangerous.

### Process pauses

A process can stop making progress temporarily because of:

-   garbage collection
-   scheduling
-   resource exhaustion
-   other runtime effects

From another machine's perspective, a paused process can look like a
failed process.

### Knowledge, truth, and lies

The system cannot always know the exact state of another component.

This is one reason distributed coordination is difficult.

### Byzantine faults

The chapter also discusses a stronger fault model in which nodes may
behave arbitrarily or maliciously, rather than merely crashing.

------------------------------------------------------------------------

## System Design --- Added: timeout ≠ failure

This is one of the most important distributed-system instincts.

``` mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: POST /charge
    Note over C: timeout
    C->>S: Retry
    S-->>C: First request may have succeeded
```

Therefore, retry design must answer:

-   Is the operation idempotent?
-   Is there a request ID?
-   Can duplicate work occur?
-   Can the server safely deduplicate?
-   Does the client know the outcome?

### Failure matrix

  Failure             Possible interpretation
  ------------------- ------------------------------
  Timeout             unknown outcome
  Connection reset    unknown outcome
  5xx                 may be retryable
  4xx                 usually request problem
  Duplicate message   normal possibility
  Delayed message     can arrive after newer state

### Practical rule

Design for **at-least-once delivery** unless you can prove stronger
semantics.

Then make consumers idempotent where possible.

------------------------------------------------------------------------

# Chapter 9 --- Consistency and Consensus

## DDIA --- Source

This chapter separates several concepts that are often mixed together.

### Linearizability

Linearizability makes a distributed system appear as though each
operation takes effect atomically at some point between invocation and
response, respecting real-time ordering.

It is a strong guarantee.

### Ordering and causality

The chapter explores:

-   ordering
-   causality
-   sequence numbers
-   total order broadcast

A system may need ordering guarantees even when it does not need every
operation to be linearizable.

### Distributed transactions

The chapter discusses atomic commit and **two-phase commit (2PC)**.

2PC coordinates participants so they agree on whether a distributed
transaction commits or aborts, but introduces coordination and
failure-handling costs.

### Consensus

Fault-tolerant consensus allows distributed nodes to agree on a value
under specified failure assumptions.

The chapter connects consensus with systems that need:

-   leader election
-   membership
-   coordination
-   consistent ordering/state

### Membership and coordination services

Coordination systems can provide primitives used by distributed
applications, but using coordination introduces its own complexity and
failure modes.

------------------------------------------------------------------------

## System Design --- Added: consistency spectrum

Do not say only "eventually consistent" or "strongly consistent."

Ask what guarantee the product actually needs.

``` text
Weaker
  |
  |  stale reads acceptable
  |  monotonic behavior
  |  causal relationships
  |  linearizable operations
  |
Stronger
```

### Example

A social-media like count may tolerate slight staleness.

A bank balance may require much stronger correctness.

A distributed lock or leader election mechanism needs coordination
properties that are different again.

### Expert question

> **What is the weakest consistency guarantee that still preserves the
> product invariant?**

Using stronger guarantees everywhere can increase latency, coordination,
and failure coupling.

------------------------------------------------------------------------

# Part III --- Derived Data

# Chapter 10 --- Batch Processing

## DDIA --- Source

Batch processing operates on accumulated data.

The chapter starts with Unix tools and the Unix philosophy, then
examines MapReduce and distributed filesystems.

### MapReduce

The programming model separates computation into:

``` text
Input
  ↓
Map
  ↓
Intermediate data
  ↓
Reduce
  ↓
Output
```

The model works well for large-scale processing because computation can
be distributed across many machines.

### Joins

The chapter examines:

-   reduce-side joins
-   grouping
-   map-side joins

The important issue is moving data to the place where computation
happens.

### Batch output

Batch jobs commonly produce derived datasets rather than directly
mutating the source system.

### Beyond MapReduce

The book discusses more general dataflow engines and the importance of
materializing intermediate state.

------------------------------------------------------------------------

## System Design --- Added: batch is for recomputation

Batch processing is particularly useful when:

-   data is large
-   latency requirements are minutes/hours rather than milliseconds
-   you need complete recomputation
-   you need periodic aggregation
-   you need to rebuild derived state

``` mermaid
flowchart LR
    A[Raw data] --> B[Batch job]
    B --> C[Derived dataset]
    C --> D[Reports]
    C --> E[Search / analytics]
```

### Important design principle

Keep authoritative data separate from derived data where practical.

If a derived index is corrupted, the ability to rebuild it from
authoritative data is extremely valuable.

------------------------------------------------------------------------

# Chapter 11 --- Stream Processing

## DDIA --- Source

Stream processing handles data that arrives continuously.

The chapter covers:

-   event streams
-   messaging systems
-   partitioned logs
-   databases and streams
-   change data capture
-   event sourcing
-   state and immutability
-   stream processing
-   time
-   stream joins
-   fault tolerance

### Messaging systems

Messages can be delivered through different mechanisms.

A key model discussed is the **partitioned log**, where events are
appended and consumers track their position.

### Change Data Capture

CDC turns database changes into a stream that other systems can consume.

This enables derived systems to react to changes in a source database.

### Event sourcing

Instead of storing only current state, the system can store a sequence
of events from which state is derived.

Conceptually:

``` text
Event 1 → Event 2 → Event 3 → Event 4
                    ↓
              Current state
```

### State, streams, and immutability

A stream of immutable events can serve as a durable history from which
different derived views are computed.

### Time

Stream processing must distinguish different notions of time, including
the time associated with events and the time at which processing occurs.

### Fault tolerance

Failures can happen during stream processing, so processing semantics
and state recovery matter.

------------------------------------------------------------------------

## System Design --- Added: event-driven architecture

A common architecture:

``` mermaid
flowchart LR
    A[Service] --> B[(Primary DB)]
    B --> C[CDC / Event Log]
    C --> D[Search Index]
    C --> E[Analytics]
    C --> F[Notification Service]
```

This separates the source of truth from derived consumers.

### But beware

Asynchronous architecture creates:

-   lag
-   retries
-   duplicate processing
-   ordering questions
-   replay requirements
-   monitoring requirements

It does **not** make consistency problems disappear. It changes where
they live.

### At-least-once processing

If a consumer can see an event more than once, design the handler to
tolerate duplicates.

Common pattern:

``` text
event_id
   ↓
deduplication / idempotent update
   ↓
derived state
```

------------------------------------------------------------------------

# Chapter 12 --- The Future of Data Systems

## DDIA --- Source

The final chapter brings together the themes of the book.

### Data integration

Real applications often combine specialized systems rather than forcing
everything into one technology.

The book discusses deriving data through:

-   batch processing
-   stream processing
-   specialized indexes
-   materialized views
-   other derived state

### Unbundling databases

The book explores the idea that a traditional database performs several
different jobs:

-   storage
-   indexing
-   querying
-   transactions
-   change propagation

These capabilities can sometimes be composed from specialized
technologies.

### Designing around dataflow

An application can be viewed as a system through which data flows:

``` text
Input
  ↓
Authoritative state
  ↓
Derived views
  ↓
Indexes / caches / analytics / models
```

### Correctness

The final chapter returns to correctness and asks where guarantees
should be enforced.

Topics include:

-   end-to-end arguments
-   constraints
-   timeliness
-   integrity
-   verification
-   predictive analytics
-   privacy and tracking

### Core conclusion

The architecture of a data-intensive application is not just a
collection of servers. It is a set of data transformations, guarantees,
and failure-handling mechanisms.

------------------------------------------------------------------------

# System Design --- Added: the derived-data architecture

A reusable architecture for many real systems:

``` mermaid
flowchart TD
    A[Clients] --> B[API / Services]
    B --> C[(System of Record)]
    C --> D[Change Stream]
    D --> E[Search Index]
    D --> F[Cache / Materialized View]
    D --> G[Analytics Pipeline]
    D --> H[Notifications]
    G --> I[Warehouse / Lake]
```

### Why this pattern is powerful

Each downstream system can be optimized for a specific access pattern.

The trade-off is that the architecture now contains multiple
representations of the same underlying facts.

Therefore:

> **Derived state must be observable, rebuildable, and understood as
> derived.**

------------------------------------------------------------------------

# Cross-Chapter Synthesis

The real value of DDIA appears when the chapters are connected.

## 1. Data model → storage engine

Your data model determines common access patterns.

Those access patterns influence storage-engine choice.

``` text
Domain model
    ↓
Queries
    ↓
Indexes
    ↓
Storage layout
    ↓
Performance
```

------------------------------------------------------------------------

## 2. Storage → partitioning

When one machine cannot handle the dataset or workload, partitioning
distributes it.

But partitioning changes query routing and transaction boundaries.

``` text
Single database
      ↓
Partition
      ↓
Distributed database
      ↓
Cross-partition queries/transactions
```

------------------------------------------------------------------------

## 3. Partitioning → replication

Partitioning solves distribution of data.

Replication solves multiple-copy availability/read scaling.

They are complementary.

``` text
             Dataset
          /            \
      Shard A          Shard B
      /    \           /    \
    R1      R2        R3      R4
```

------------------------------------------------------------------------

## 4. Replication → consistency

Once multiple copies exist, the system must define what readers are
allowed to observe.

This leads naturally to:

-   replication lag
-   read-your-writes
-   monotonic reads
-   consistent prefixes
-   stronger consistency guarantees

------------------------------------------------------------------------

## 5. Transactions → distributed systems

A transaction is straightforward when all data is local.

When state crosses partitions or services, coordination becomes harder.

This leads toward:

-   distributed transactions
-   2PC
-   consensus
-   asynchronous workflows
-   idempotency
-   compensation

------------------------------------------------------------------------

## 6. Batch → stream

Batch answers:

> "What should we compute from the data we have accumulated?"

Stream processing answers:

> "What should we do as new data arrives?"

Many production architectures use both.

``` text
             Events
                |
        +-------+-------+
        |               |
      Stream           Batch
        |               |
   Low latency       Recompute
        |               |
        +-------+-------+
                |
         Derived state
```

------------------------------------------------------------------------

# The System Design Mental Model

When given a system-design problem, walk through these layers.

## Layer 1 --- Requirements

Write:

-   functional requirements
-   scale
-   latency
-   availability
-   durability
-   consistency
-   geographic requirements

## Layer 2 --- APIs

Define the major operations.

Example:

``` text
POST /items
GET  /items/{id}
GET  /users/{id}/items
```

Do not design infrastructure before knowing the operations.

## Layer 3 --- Data model

Identify:

-   entities
-   relationships
-   access patterns
-   indexes
-   immutable vs mutable data

## Layer 4 --- Baseline architecture

Start simple.

``` mermaid
flowchart LR
    A[Clients] --> B[Load Balancer]
    B --> C[Application]
    C --> D[(Database)]
    C --> E[Cache]
```

Then ask what breaks.

## Layer 5 --- Scale reads

Options may include:

-   caching
-   read replicas
-   materialized views
-   search indexes

## Layer 6 --- Scale writes

Options may include:

-   batching
-   asynchronous processing
-   partitioning
-   workload isolation

## Layer 7 --- Handle failures

Ask:

-   What if DB fails?
-   What if a replica lags?
-   What if a request times out?
-   What if a message is duplicated?
-   What if a consumer is down?
-   What if a region disappears?

## Layer 8 --- Correctness

Identify invariants.

Examples:

``` text
A payment must not be charged twice.
A username must be unique.
A completed order must not disappear.
A message may be delivered more than once.
```

Then choose mechanisms that preserve them.

## Layer 9 --- Observability

Monitor:

-   latency
-   throughput
-   errors
-   saturation
-   queue lag
-   replication lag
-   cache hit rate
-   storage growth
-   failed jobs
-   consumer offsets

## Layer 10 --- Operations

Ask:

> "How do we operate this system for years?"

That question connects directly to DDIA's emphasis on operability,
simplicity, and evolvability.

------------------------------------------------------------------------

# High-Value Trade-offs to Master

## Availability vs consistency

More coordination can produce stronger guarantees but can increase
latency or reduce availability under failures.

## Latency vs durability

Waiting for durable replication can increase write latency.

## Read performance vs write cost

Indexes and materialized views speed reads but add write/update work.

## Simplicity vs specialization

One general-purpose database is operationally simpler.

Multiple specialized systems may serve workloads better but increase
system complexity.

## Normalization vs duplication

Normalization reduces duplication and update anomalies.

Duplication can make read paths faster and simpler.

## Synchronous vs asynchronous

Synchronous paths make dependencies explicit and can provide stronger
guarantees.

Asynchronous paths improve decoupling and resilience but introduce lag
and eventual convergence.

## Automatic vs manual rebalancing

Automation reduces operator work but must itself be reliable and
understandable.

------------------------------------------------------------------------

# Common System Design Mistakes

## Mistake 1 --- Technology-first design

Bad:

> "Let's use Kafka, Redis, Cassandra, and Elasticsearch."

Better:

> "The feed requires high write throughput, asynchronous fan-out,
> low-latency reads, and a rebuildable derived view."

Then select technologies.

## Mistake 2 --- Calling everything "scalable"

Always specify:

-   read scaling
-   write scaling
-   storage scaling
-   geographic scaling
-   organizational/operational scaling

## Mistake 3 --- Ignoring failure semantics

A timeout is not proof of failure.

A retry can create a duplicate.

A message can be processed twice.

## Mistake 4 --- Treating caches as source of truth

If possible, make cached/derived data rebuildable.

## Mistake 5 --- Using strong consistency everywhere

Use the guarantee the business actually requires.

## Mistake 6 --- Forgetting rebalancing

A distributed database that works with 3 nodes may have a very different
operational problem when moving data between 30 or 300 nodes.

## Mistake 7 --- Ignoring schema evolution

Production systems rarely upgrade every component simultaneously.

## Mistake 8 --- Designing only the happy path

For every major arrow in an architecture, ask:

> What happens when this arrow breaks?

------------------------------------------------------------------------

# Practical Design Patterns Derived from DDIA Concepts

## Pattern A --- Read-heavy service

``` mermaid
flowchart LR
    A[Client] --> B[API]
    B --> C{Cache}
    C -->|Hit| A
    C -->|Miss| D[(DB)]
    D --> C
    C --> A
```

Use when repeated reads dominate and stale data is acceptable within the
product requirements.

------------------------------------------------------------------------

## Pattern B --- Read replicas

``` mermaid
flowchart LR
    A[Application] --> L[(Leader)]
    L --> R1[(Replica)]
    L --> R2[(Replica)]
    A --> R1
    A --> R2
```

Useful for read scaling when replication lag is acceptable.

------------------------------------------------------------------------

## Pattern C --- Async work queue

``` mermaid
flowchart LR
    A[API] --> B[Queue]
    B --> C[Worker 1]
    B --> D[Worker 2]
    B --> E[Worker N]
```

Useful when work does not need to finish during the user's request.

------------------------------------------------------------------------

## Pattern D --- Event-driven derived state

``` mermaid
flowchart LR
    A[(Source DB)] --> B[Change Stream]
    B --> C[Search]
    B --> D[Analytics]
    B --> E[Notifications]
```

Useful when multiple specialized consumers need to react to the same
changes.

------------------------------------------------------------------------

## Pattern E --- Partitioned service

``` mermaid
flowchart LR
    A[Router] --> B[Shard 1]
    A --> C[Shard 2]
    A --> D[Shard 3]
    A --> E[Shard N]
```

Useful when one node cannot handle the complete dataset/workload.

------------------------------------------------------------------------

# Worked Thinking Example --- Design a Social Feed

This example is **System Design --- Added**. It is not presented as a
direct reconstruction of the book.

## Requirements

Assume:

-   users follow other users
-   users publish posts
-   users read a personalized feed
-   feed reads should be fast
-   occasional slight staleness is acceptable

## Start simple

``` mermaid
flowchart LR
    A[Client] --> B[Feed API]
    B --> C[(Posts DB)]
```

## Identify bottleneck

A feed may require combining posts from many followed users.

Repeatedly calculating that at request time can become expensive.

## Derived feed

``` mermaid
flowchart LR
    A[Post Service] --> B[(Posts)]
    A --> C[Event Stream]
    C --> D[Feed Workers]
    D --> E[(Feed Store)]
    F[Client] --> G[Feed API]
    G --> E
```

Now feed reads are cheap, but writes produce more asynchronous work.

## New failure modes

Ask:

-   What if feed worker is down?
-   What if events are duplicated?
-   What if a feed entry is stale?
-   What if a celebrity has millions of followers?
-   Can the feed be rebuilt?

This is the DDIA mindset: every optimization introduces new state and
new failure modes.

------------------------------------------------------------------------

# Expert-Level Questions by Chapter

## Chapter 1

-   What exactly is the load?
-   What is the performance target?
-   What failure modes matter?
-   How will operators diagnose problems?

## Chapter 2

-   What is the data model?
-   What are the dominant queries?
-   Where are the relationships?
-   What is the natural aggregate?

## Chapter 3

-   What is the storage engine optimized for?
-   How do indexes affect writes?
-   Is this OLTP or analytics?
-   Where does amplification occur?

## Chapter 4

-   Can old and new versions coexist?
-   What happens during rolling deployments?
-   Who owns schema compatibility?

## Chapter 5

-   Where are replicas?
-   What is the replication lag?
-   What consistency can readers observe?
-   What happens during failover?

## Chapter 6

-   What is the partition key?
-   Can it create hot spots?
-   How is data rebalanced?
-   How are requests routed?

## Chapter 7

-   What invariant requires atomicity?
-   What isolation level is necessary?
-   What happens under concurrent updates?

## Chapter 8

-   What does a timeout mean?
-   Can requests be duplicated?
-   What does the system know vs merely assume?
-   What happens if clocks disagree?

## Chapter 9

-   What consistency guarantee is required?
-   Do we need ordering?
-   Do we need consensus?
-   Where is coordination unavoidable?

## Chapter 10

-   Can the result be recomputed?
-   How is intermediate state handled?
-   What happens when a batch job fails?

## Chapter 11

-   What is the event model?
-   What delivery semantics exist?
-   How is state recovered?
-   What is event time vs processing time?

## Chapter 12

-   Which data is authoritative?
-   Which data is derived?
-   Can derived state be rebuilt?
-   Where are correctness guarantees enforced?

------------------------------------------------------------------------

# 12-Chapter Learning Roadmap

## Phase 1 --- Foundations

### Week 1

Chapter 1 --- Reliability, scalability, maintainability

Practice: - design a URL shortener - identify load parameters - list
failure modes

### Week 2

Chapter 2 --- Data models

Practice: - model a messaging app - compare relational vs document vs
graph approaches

### Week 3

Chapter 3 --- Storage

Practice: - explain B-tree vs LSM-tree - design indexes for several
query patterns

### Week 4

Chapter 4 --- Encoding/evolution

Practice: - design a versioned API - design an event schema that
survives rolling deployments

------------------------------------------------------------------------

## Phase 2 --- Distributed data

### Week 5

Chapter 5 --- Replication

Practice: - leader/follower design - read-after-write problem - failover
analysis

### Week 6

Chapter 6 --- Partitioning

Practice: - choose partition keys - identify hot spots - explain
rebalancing

### Week 7

Chapter 7 --- Transactions

Practice: - payment transfer - inventory reservation - concurrent
booking

### Week 8

Chapter 8 --- Distributed failure

Practice: - timeout/retry scenarios - duplicate requests - clock/failure
thought experiments

### Week 9

Chapter 9 --- Consistency/consensus

Practice: - choose consistency guarantees - explain leader election -
explain why distributed coordination is expensive

------------------------------------------------------------------------

## Phase 3 --- Dataflow

### Week 10

Chapter 10 --- Batch

Practice: - analytics pipeline - daily aggregation - rebuild a derived
dataset

### Week 11

Chapter 11 --- Streams

Practice: - event-driven order processing - CDC pipeline - idempotent
consumer

### Week 12

Chapter 12 --- Integration

Practice: - design a complete data-intensive application - identify
authoritative vs derived data - document every failure path

------------------------------------------------------------------------

# Final Mental Checklist

Before calling a system design "done", answer these.

### Requirements

-   What does the system do?
-   What does it not do?
-   What scale is required?

### Data

-   What is authoritative?
-   What is derived?
-   What are the access patterns?

### Performance

-   Where is the hot path?
-   What is cached?
-   What is precomputed?

### Distribution

-   What is partitioned?
-   What is replicated?
-   How is routing performed?

### Correctness

-   Which invariants matter?
-   Which operations require atomicity?
-   What consistency guarantee is sufficient?

### Failure

-   What happens when a node dies?
-   What happens when a network call times out?
-   What happens when a message is duplicated?
-   What happens during recovery?

### Evolution

-   Can schemas change safely?
-   Can services deploy independently?
-   Can derived state be rebuilt?

### Operations

-   What do we monitor?
-   How do we debug lag?
-   How do we recover?
-   How do we rebalance?
-   How do we roll back?

------------------------------------------------------------------------

# One-Page DDIA Cheat Sheet

  -----------------------------------------------------------------------
  Concept                             Mental model
  ----------------------------------- -----------------------------------
  Reliability                         Keep working correctly despite
                                      faults

  Scalability                         Handle increasing load while
                                      maintaining acceptable performance

  Maintainability                     Make operation, understanding, and
                                      change manageable

  Data model                          Shape data around domain + access
                                      patterns

  Index                               Additional structure that makes
                                      particular queries faster

  LSM-tree                            Log-structured writes + sorted
                                      files + compaction

  B-tree                              Page-oriented ordered tree with
                                      in-place updates

  Replication                         Multiple copies of data

  Partitioning                        Split data across nodes

  Replication lag                     Followers temporarily behind the
                                      leader

  Transaction                         Group operations under correctness
                                      guarantees

  Isolation                           Controls effects of concurrent
                                      transactions

  Partial failure                     Some components fail while others
                                      continue

  Linearizability                     Operations appear atomically
                                      ordered in real time

  Consensus                           Nodes agree despite specified
                                      failures

  Batch                               Process accumulated data

  Stream                              Process continuously arriving
                                      events

  CDC                                 Turn database changes into a stream

  Event sourcing                      Store events and derive state

  Materialized view                   Persist derived query results

  Derived data                        Data computed from
                                      authoritative/source data

  Idempotency                         Repeating an operation has the same
                                      intended effect

  Schema evolution                    Change data formats without
                                      breaking readers/writers
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Final Takeaway

The most important lesson to carry from DDIA into system design is not a
list of technologies.

It is a way of thinking:

``` text
                    ┌───────────────┐
                    │ Requirements  │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │ Data + Access │
                    │   Patterns    │
                    └───────┬───────┘
                            ↓
               ┌────────────┴────────────┐
               ↓                         ↓
          Storage                    Processing
               ↓                         ↓
          Partitioning               Batch/Stream
               ↓                         ↓
          Replication               Derived State
               └────────────┬────────────┘
                            ↓
                  Consistency + Correctness
                            ↓
                    Failure Handling
                            ↓
                  Operations + Evolution
```

When you become comfortable moving through those questions, system
design stops being a memorization exercise.

You are no longer asking:

> "Which architecture did someone else use?"

You are asking:

> **"Given this workload, these invariants, these failure assumptions,
> and these operational constraints, what architecture follows---and
> what trade-offs am I accepting?"**

That is the core skill this guide is designed to build.

------------------------------------------------------------------------


