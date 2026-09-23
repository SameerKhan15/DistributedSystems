# Epochs in Distributed Systems

## Stage 1 — Why Do We Need Epochs?

### The Core Problem

In a distributed database, completing a transaction locally does **not**
necessarily mean that the transaction is globally visible.

For example:

    Initial state:

    Node A visibility = 100
    Node B visibility = 100
    Node C visibility = 100

A client writes version 101 on Node A.

After the local commit completes, the system may temporarily be:

    Node A visibility = 101
    Node B visibility = 101
    Node C visibility = 100

Therefore:

    Local commit != Global visibility

If Node A acknowledges the write at this point, a subsequent read could be
routed to Node C and return version 100.

If the system promises cross-node read-after-write consistency, this would
violate that contract.

---

### Cross-Node Read-After-Write

Suppose a write completes before a subsequent read begins:

    WRITE(101) completes
             |
             v
            ACK
             |
             v
       READ on another node

The later read must not observe a version older than 101.

Conceptually:

    WRITE(V) completed
        =>
    subsequent READ >= V

Therefore, a write acknowledgement may represent more than local durability.

It may need to mean:

    "The distributed system has advanced sufficiently that a subsequent
     operation routed to another node will not fall behind this write."

---

### Overlapping Operations Are Different

Consider:

    WRITE(101)
    |---------------------|
             READ
             |------|

The read overlaps the write.

If the read returns version 100, this is not necessarily a consistency
violation because the write had not yet completed when the read began.

There is no established real-time ordering of the form:

    WRITE_complete < READ_start

Contrast this with:

    WRITE(101)
    |---------| ACK

                  READ
                  |-----|

Now:

    WRITE_complete < READ_start

If cross-node read-after-write consistency is guaranteed, the later read
must observe version 101 or something newer.

---

### Monotonic Reads

A related issue occurs between reads.

Suppose:

    READ #1 -> version 101

and a later read is routed to another node:

    READ #2 -> version 100

The client has observed the system move backward.

A monotonic-read guarantee requires:

    READ(V)
       =>
    later READ >= V

---

### Distributed Safe Visibility Frontier

Suppose nodes have progressed to:

    Node A = 108
    Node B = 105
    Node C = 107

The highest version known to be safe across the entire cluster is:

    safe_frontier = min(108, 105, 107)
                  = 105

Therefore:

    <= 105 : known to be safe everywhere
    >  105 : may only be visible on a subset of nodes

General model:

    SafeVisibilityFrontier =
        min(per-node visibility frontiers)

This introduces an important distributed-systems distinction:

    Something being true on one node
        !=
    that node knowing it is globally true

A coordination protocol is required to establish that all relevant
participants have crossed a particular logical point.

---

### Performance Engineering View

Suppose:

    local commit time   = 1 ms
    total write latency = 15 ms

The remaining 14 ms may not simply be "miscellaneous overhead."

It may represent:

    local work
        |
        v
    distributed coordination / visibility barrier
        |
        v
    safe acknowledgement

Optimizing local I/O from:

    1 ms -> 0.5 ms

would barely improve total latency if distributed coordination dominates
the request's critical path.

Therefore, when analyzing latency, ask:

    What correctness condition is this request waiting for?

Is it waiting for:

    - computation?
    - I/O?
    - durability?
    - replication?
    - distributed propagation?
    - a synchronization barrier?

---

### Slowest-Participant Effect

If cluster-wide progress requires every participating node to cross a
frontier:

    Node A = 1 ms
    Node B = 2 ms
    Node C = 15 ms

then barrier completion may be approximately determined by:

    max(1, 2, 15) = 15 ms

Distributed coordination latency is therefore often determined by the
slowest participant rather than the average participant.

This makes events such as:

    - network delay
    - CPU saturation
    - storage stalls
    - runtime pauses
    - noisy-neighbor effects

potential contributors to cluster-wide tail latency.

---

## Stage 1 Key Invariants

1. Local commit does not necessarily imply global visibility.

2. Requests may move between nodes with different visibility frontiers.

3. Once a write is acknowledged, a later cross-node read must not go
   backward if the system promises read-after-write consistency.

4. The writer may therefore need evidence of sufficiently global progress
   before safely returning an acknowledgement.

5. A useful abstraction is a distributed safe visibility frontier.

6. Global progress may be bounded by the slowest participating node.

7. Distributed coordination can therefore sit directly on the request's
   latency critical path.


# Stage 2 — Epochs and Logical Transaction Versions

## Core Idea

An epoch can be viewed as a logical time window or generation during which
multiple transactions receive ordered logical version numbers.

Conceptually:

    Epoch N

        transaction version N:1
        transaction version N:2
        transaction version N:3
        ...
        transaction version N:k

An epoch therefore provides a coarse-grained grouping mechanism, while
transactions still receive a finer-grained ordering within that epoch.

---

### Epoch vs Transaction Version

An epoch is not itself necessarily equivalent to a single transaction
version.

Instead:

    Epoch
      =
    logical grouping / generation

while:

    Transaction Version
      =
    individual logical position within that generation

A useful conceptual representation is:

    LogicalVersion =
        EpochIdentifier + PerEpochSequence

The exact encoding is implementation-dependent.

---

### Why Have a Per-Epoch Sequence?

Multiple transactions can commit during the same epoch.

They still need distinct logical positions.

For example:

    Epoch N

    T1 -> N:1
    T2 -> N:2
    T3 -> N:3

This gives:

    T1 < T2 < T3

while preserving the fact that all three belong to the same epoch.

Therefore:

    Epoch      = coarse-grained grouping
    Sequence   = fine-grained ordering

---

### Bounded Version Space

Some implementations allocate only a bounded number of transaction sequence
values within an epoch.

Conceptually:

    Epoch N

    sequence 1
    sequence 2
    ...
    sequence K

If the available sequence space is exhausted before a new epoch opens,
additional transactions may need to wait for the system to advance to the
next epoch.

This means an epoch can serve not only as a consistency abstraction but
also as an allocation boundary.

---

### Performance Engineering View

Suppose:

    maximum transactions per epoch = K
    epoch duration                 = T

Then a rough sequence-space throughput ceiling is:

    R_max ~= K / T

This is not necessarily the actual system throughput limit.

Other bottlenecks may dominate first, such as:

    - CPU
    - storage
    - network
    - replication
    - locking
    - synchronization
    - scheduler capacity

But bounded logical-version space can still create a hidden saturation
point under sufficiently high transaction rates.

---

## Stage 2 Mental Model

1. An epoch is a logical time window or generation.

2. Multiple transactions can receive distinct logical versions within the
   same epoch.

3. A transaction version can be thought of conceptually as:

       EpochIdentifier + PerEpochSequence

4. The epoch provides coarse-grained grouping.

5. The sequence provides fine-grained ordering within the group.

6. Some implementations use bounded per-epoch sequence space.

7. Exhausting that space can force transactions to wait for another epoch.

8. Epochs therefore provide both:

       grouping / batching
       +
       ordered transaction identity


# Stage 3 — Commit Progress vs Visibility Progress

## Core Idea

A distributed database may maintain two different logical frontiers:

    CommitAssignmentFrontier
    SafeVisibilityFrontier

They represent different forms of progress.

    CommitAssignmentFrontier
        = how far logical transaction assignment / commit activity
          has progressed

    SafeVisibilityFrontier
        = how far readers are currently allowed to observe safely

At the beginning of an epoch, the two frontiers may be aligned.

Example:

    SafeVisibilityFrontier    = 5000
    CommitAssignmentFrontier  = 5000

As transactions receive logical versions:

    T1 -> 5001
    T2 -> 5002
    T3 -> 5003

the commit-assignment frontier advances while the visibility frontier may
remain fixed:

    SafeVisibilityFrontier    = 5000
    CommitAssignmentFrontier  = 5003

This gap can be intentional.

---

### Committed Does Not Necessarily Mean Visible

A transaction may already have:

    - an assigned logical version
    - completed local durability work

and still not yet be visible to new readers.

Example:

    T1 version              = 5001
    SafeVisibilityFrontier  = 5000

T1 may be committed, but its version is still ahead of the reader-safe
visibility frontier.

Therefore:

    Committed != Visible

This distinction is fundamental in many distributed systems.

There may be several different notions of completion:

    work assigned
        |
        v
    locally completed
        |
        v
    durable
        |
        v
    globally / safely visible

These states should not be conflated.

---

### The Batching Window

Suppose:

    SafeVisibilityFrontier = 5000

and transactions receive:

    T1 = 5001
    T2 = 5002
    T3 = 5003
    T4 = 5004

Conceptually:

    5000 | 5001 5002 5003 5004
         |
         +--- reader-safe frontier

The transactions to the right of the visibility frontier represent newer
logical work that has not yet crossed the safe-reader boundary.

The gap between:

    SafeVisibilityFrontier

and:

    CommitAssignmentFrontier

can form a batching window.

Instead of advancing visibility after every transaction, a system may
accumulate multiple transactions and make them visible together.

---

### Epoch Advance Can Close the Gap

Suppose a later epoch begins at logical point:

    6000

The system may then advance both logical frontiers:

    CommitAssignmentFrontier = 6000
    SafeVisibilityFrontier   = 6000

Now:

    T1 = 5001
    T2 = 5002
    T3 = 5003
    T4 = 5004

are all behind the new visibility frontier.

Conceptually:

    Epoch opens
        |
        v
    frontiers aligned
        |
        v
    transactions receive logical versions
        |
        v
    commit frontier advances
    visibility frontier stays fixed
        |
        v
    batching window grows
        |
        v
    epoch advances
        |
        v
    safe visibility frontier advances
        |
        v
    earlier transactions become visible together

---

### Why Hold Visibility Back?

If the visibility frontier advanced after every transaction:

    T1 commits -> advance visibility
    T2 commits -> advance visibility
    T3 commits -> advance visibility

the system could repeatedly pay synchronization, coordination, and
wake-up costs.

Instead, a system may batch transactions within a logical window and
advance visibility once for the group.

This trades:

    improved batching
    lower synchronization overhead
    better throughput efficiency

for:

    additional waiting latency for individual transactions

This is a classic throughput-vs-latency tradeoff.

---

### Performance Engineering View

Transaction latency may not be determined only by local commit or storage
latency.

A transaction can enter a state such as:

    local commit complete
        |
        v
    waiting for visibility frontier
        |
        v
    visible / releasable

Therefore, useful performance questions include:

    - How long do transactions spend waiting after local commit?
    - How frequently does the safe visibility frontier advance?
    - How large does the gap between commit progress and visibility progress
      become?
    - Is latency dominated by local work or by waiting for coordination?
    - How much throughput benefit is gained from batching?
    - Does one slow participant delay advancement of the visibility frontier?
    - How does the batching interval affect tail latency?

---

## Stage 3 Key Invariants

1. Commit progress and visibility progress are different concepts.

2. A system may intentionally allow commit activity to advance ahead of
   reader-visible state.

3. A transaction can be committed but not yet visible.

4. The gap between commit progress and visibility progress can act as a
   batching window.

5. Advancing the visibility frontier can make many earlier transactions
   visible together.

6. Batching amortizes synchronization cost across multiple transactions.

7. The tradeoff is improved throughput efficiency at the cost of some
   additional latency.

8. For performance analysis, the important question is often not merely
   "How long did the commit take?" but:

       "What progress condition is the request waiting for before it
        becomes safe to release?"  



