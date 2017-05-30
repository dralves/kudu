---
layout: post
title: Consistency in Apache Kudu, Part 1
author: David Alves
---
In this series of short blog posts we will introduce Kudu’s consistency model,
its design, ultimate goals, what is currently implemented, and what are the next steps.
On the way, we’ll shed some light into some of the more relevant components and how they
fit together. In this post we’ll cover motivation, overall architecture, and read/write
paths. In follow-up posts we’ll dive into particular components and wrap up with the big picture
of how they all play together and what are our future plans.

In Part 1 of the series (this one), we review/introduce some concepts, motivation and the relevant
Kudu components that play a part in consistency.

<!--more-->

## What is “consistency” and why is it relevant?

Modern storage systems like Kudu have to support many concurrent users while coordinating requests
across many machines, each with many threads executing work at the
same time. However, application developers shouldn’t have to understand the internal
details of how these systems implement this parallel, distributed, execution in order to write
correct applications. Consistency in the context of parallel, distributed systems usually
refers to how the system behaves in comparison to single thread, single machine system.
In such a system operations would happen one-at-a-time, in a clearly defined order,
making correct applications easy to code and reason about. A developer
using such a storage system doesn’t have to care about ordering anomalies, so the code is
simpler, but more importantly, cognitive load is greatly reduced, freeing focus for other,
more important, things.

While such a single-threaded, single-machine storage system is definitely possible to build,
it wouldn’t be able to cope with very large amounts of data. In order to deal with big data
volumes and write throughputs modern storage systems like Kudu re designed to be distributed,
storing and processing data across many machines and cores, which means that many things happen
simultaneously in different locations and that there are more pieces that can fail.
This means having to deal with data inconsistencies and corruption or prevent them altogether.
How far systems go (or don’t go) in emulating the single-threaded, single-machine system in this
distributed, parallel setting 'where failures are common and many things happen at the same time'
is roughly what is referred to as how “consistent” the system is.

Consistency as a term is somewhat overloaded in the distributed systems and database communities,
there are many different models, properties, different names for the same concept, and often
different concepts under the same name. This post is not meant to introduce these concepts
as there are excellent references already available [elsewhere](https://aphyr.com/posts/313-strong-consistency-models).
Throughout the post we’ll refer to consistency loosely as the C in CAP[ref]
in some cases and as the I in ACID[ref] in others; we’ll try to be specific when relevant.

## Design decisions, trade-offs and motivation

Consistency is essentially about ordering and ordering usually has a cost. Distributed storage
system design must choose to prioritize some properties over others according to the target use
cases. That is, trade-offs must be made or, borrowing a term from machine learning, there is
“no free lunch”. Different systems choose different trade-offs points, for instance, when
discussing Dynamo[ref] inspired systems, engineers usually talk about the consistency/availability
trade-off: by allowing a write to a data item to succeed even when a majority (or even all) of the
replicas serving that data item are unreachable, Dynamo’s design is minimizing insertion errors and
insert latency at the cost having to perform extra work for value reconciliation on reads and
possibly returning stale or disordered values. On the other end of the spectrum, traditional DBMS
design is often driven by the need to support transactions of arbitrary complexity while providing
the users stronger, predictable, semantics. This usually comes at the cost of scalability,
availability and overall performance.

Kudu’s overarching goal is to enable very fast analytic over large amounts of data, meaning it was
designed to perform fast scans that go over large volumes of data stored in many servers.
In practical terms this means that, when given a choice, more often than not, we opted for the
design that would enable Kudu to have faster scan performance, i.e. reads, even if it meant pushing
a bit more work to the path that mutates data, i.e. writes. This does not mean that the write path
was not a concern altogether. In fact, modern storage systems like Google’s Spanner[ref]
global-scale database demonstrate that, with right set of trade-offs, it is possible to have strong
consistency semantics with write latencies and overall availability that are adequate for most use
cases. For the write path, we often made similar choices in Kudu.

Another important aspect that informed design decisions is the type of write workload we targeted.
Traditionally, analytical storage systems target periodic bulk write workloads and a continuous
stream of analytical scans. This design is often problematic in that it forces users to have to
build complex pipelines where data is accumulated in one place for later loading into the storage
system. Moreover, beyond the architectural complexity factor, this kind of design also
means that the data that is available for analytics is not the most recent. In Kudu we aimed for
enabling continuous ingest, i.e. having a continuous stream of small writes, obviating the need to
assemble a pipeline for data accumulation/loading and allowing analytical scans to have access to
the most recent data. Another important aspect of the write workloads that we targeted in Kudu is
that they are append-mostly, i.e. most insert new values into the table, with a smaller percentage
updating currently existing values. Both the average write size and the data distribution influence
the design of the write path, as we’ll see in the following sections.

One last concern we had in mind is that different users have different needs when it comes to
consistency semantics, particularly as it applies to an analytical storage system like Kudu. For
some users consistency isn’t a primary concern, they just want fast scans, and the ability to
update/insert/delete values without needing to build a complex pipeline. For example, many machine
learning models are insensitive to data recency or ordering so, when using Kudu to store data that
will be used to train such a model, consistency is often not as primary a concern as performance is.
In other cases consistency is much higher in the priority scale. For example, when using Kudu to
store transaction data for fraud analysis it might be important to capture if events are causally
related. Fraudulent transactions might be characterized by a specific sequence of events and when
retrieving that data it might be important for the scan result to reflect that sequence. Kudu’s
design allows users to make a trade off between consistency and performance at scan time. That is,
users can choose to have stronger consistency semantics or scans at the penalty of latency and
throughput or they can choose to weaken the consistency semantics for an extra performance boost.

### Note

Kudu currently lacks support for atomic multi-row mutation operations (i.e. mutation
operations to more than one row in the same or different tablets, planned as a future feature) so,
when discussing writes, we’ll be talking about the consistency semantics of single row mutations.
In this context we’ll discuss Kudu’s properties more from a key/value store standpoint. On the
other hand Kudu is an analytical storage engine so, for the read path, we’ll also discuss the
semantics of large (multi-row) scans. This moves the discussion more into the field of traditional
DBMSs. These ingredients make for a non-traditional discussion that is not exactly apples-to-apples
with what the reader might be familiar with, but our hope is that it still provides valuable, or
at least interesting, insight.

## Relevant architecture components and scope:

In follow up posts we’ll cover these components and the role they play regarding consistency in
more detail, but it might be useful to introduce them briefly before covering the write and read
paths. Kudu currently lacks support for atomic multi-row mutation operations (i.e. mutation
operations to more than one row in the same or different tablets, planned as a future feature) so,
for the write path, we’ll be talking about the consistency semantics of single row mutations.
In this context we’ll discuss Kudu’s properties more from a key/value store standpoint. On the
other hand Kudu is an analytical storage engine so, for the read path, we’ll also discuss the
semantics of large (multi-row) scans. This moves the discussion more into the field of traditional
DBMSs. These ingredients make for a non-traditional discussion that is not exactly apples-to-apples
with what the reader might be familiar with, but our hope is that it still provides valuable, or
at least interesting, insight.

The components that are relevant from a consistency stand point are:

* Operation execution: Client write requests are accepted by a leader tablet replica and submitted
for processing. A component called “TransactionDriver” is responsible for executing each operation
to completion, meaning executing each of the phases we’ll cover in the [ref anatomy of  a write section].
* Consensus: Each tablet that is part of a table is served by multiple replicas. Write requests are
replicated to follower replicas by a leader replica following the Raft consensus algorithm[ref].
For each tablet, both leader and follower replicas observe the same order of requests. When the
leader becomes unavailable a new leader is transparently appointed and clients move to the new
leader to continue writing. All replicas are able to serve scans.
* Timestamp assignment and propagation: Each write request is assigned a per-tablet monotonically
increasing timestamp, assigned by the consensus component, that is a hybrid between a physical
timestamp and a logical timestamp. This fact that this timestamp has a physical component enables
point-in-time queries. Timestamp monotonicity is automatically enforced both for writes to the same
tablet and for writes from the same client to multiple tablets. This means that, for each client,
the causal relationship between both reads and writes is captured, i.e. that clients can
read-their-writes. Users have the choice to enforce this monotonicity even across clients as we’ll
see in future posts.
* WAL: Each replica stores client requests in the Write-Ahead-Log (WAL), after ordering has been
established by the leader replica. A request is not considered “committed” until is has landed on
the WAL of a majority of replicas. The WAL also stores the in-memory data stores that were mutated
by any given client request. This component serves a dual purpose storing the original client
requests and how they mutated each tablet’s internal state after being applied.
* MVCC: The Multi-Version Concurrency Control system tracks which operations are in-flight and
which have been committed to make sure that unfinished operations are not visible to clients until
they complete. Scans take a snapshots of MVCC at the time they start executing and use this snapshot
to filter out unfinished operations as a tablet is scanned.
* Replay Cache: Each write request from a client is executed exactly once by a tablet’s replicas,
this is coordinated by the ReplayCache component. This guarantee applies even when the leader of a
tablet changes and across restarts as it relies on metadata that is made durable along with the
client’s request.

## Anatomy of a write:

As we mentioned previously in this post series we’ll be considering single-row mutations.
In this post in particular we won’t be covering fault/partition tolerance and recovery, we’ll just
be covering the “happy” path, in stable state. 

Writes in Kudu go through the following steps:

1. Submission: A client wanting to perform a row mutation submits a write to the leader tablet.
Only leader tablets accept client mutation requests.
2. Prepare: The tablet server receives the request and submits it for prepare. The prepare phase of
an operation is executed serially, a single request at a time.
    1. Lock acquisition: The first step in the prepare phase of an operation is lock acquisition.
    The server decodes the client’s request and acquires a lock for each of the rows that might be
    mutated. This makes sure that following operations that want to mutate the same set of rows have
    to wait until this operation has completed.
    2. Timestamp assignment: The second step in the prepare phase is to assign a timestamp to the
    operation. We’ll provide more details on exactly what this timestamp is and how Kudu enforces
    the properties we’re referring to in follow up posts, but for now the reader can just think of
    it as a per-tablet monotonically increasing number. That timestamp enables Kudu to establish
    ordering between any two related operations.
    3. Register the operation with MVCC: Once the timestamp is assigned, it is registered with the
    Multi-Version Concurrency Control mechanism. This makes sure that readers are aware of which
    operations are in-flight and they don’t return u. We’ll discuss this in more detail when we
    cover the read path.
    4. Submit the operation for replication through consensus: The final step in the prepare phase
    is to submit the operation to consensus to be replicated to non-leader replicas. Upon receiving
    the operation follower replicas will execute the prepare phase in the exact same sequence as the
    leader replicas did. The leader replica writes the clients’s request, as is, to
    the write-ahead-log.
3. Apply: Upon successful completion of the the prepare phase, i.e. when consensus has established
that an operation has been replicated to and accepted by a majority of replicas, the operation is
ready to be applied to the tablet. Note that until this point tablet data has not been mutated.
    1. Mutate tablet entries: Each of the mutations specified by the client are now applied to the
    tablet. We’ll refer the reader back to the Kudu paper [ref] for more information about exactly
    which data structures are mutated and how this is done. For this blog post the relevant aspects
    of this are: a) contrary to the Prepare phase this phase is multi-threaded; b) in the large
    majority of cases apply only changes in-memory data structures. Regarding a) since operations
    that mutate the same rows only move to the apply phase one at a time due to lock acquisition
    (step 2.1) it is now safe to have multiple operations be executed at the same time, since we’re
    sure that each will mutate different sets of rows. Regarding b) since the operation was appended
    to consensus we’re sure that it’ll survive a crash, so there is no need (from a correctness stand
    point) to write any of the actual tablet changes to disk.
    2. Write an operation commit entry: During step 3.1, for each row, unique identifiers of the data
    structures that were changed were collected. This write doesn’t need to be made durable before a
    response is sent back to the client, though there are some constraints. The commit entry is
    crucial for crash recovery as we’ll see in follow up posts. 
    3. Mark the operation as committed in MVCC: The final step before replying to the client is to
    mark the operation as committed in MVCC. The changes performed by the operation will become
    visible to reader.
    4. Reply to the client: Once every one of the previous steps is completed the response is sent
    to the client.

## References


