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

# Stage 4 — Single-Node Epoch Lifecycle  
## Core Idea  
Within a single node, a transaction can move through several distinct states:

    logical version assigned
        |
        v
    locally committed / durable
        |
        v
    committed but not yet visible
        |
        v
    visibility frontier advances
        |
        v
    visible / releasable

The important point is:

    Assigned != Committed != Visible != Released

These are different stages of progress.  

## Example

Suppose a logical epoch begins with:

    SafeVisibilityFrontier   = 5000
    CommitAssignmentFrontier = 5000

Transaction T1 begins committing and receives:

    T1 -> 5001

After its local durability work completes:

    T1 version = 5001
    SafeVisibilityFrontier = 5000

Since:

    5001 > 5000

T1 is:

    COMMITTED
    but
    NOT YET VISIBLE

A transaction can therefore be durable without yet being exposed to new
readers.  

## Multiple Transactions Can Accumulate

While the visibility frontier remains fixed, additional transactions may
commit:

    T1 -> 5001
    T2 -> 5002
    T3 -> 5003

Conceptually:

    5000 | 5001 5002 5003
         |   T1   T2   T3
         |
         +--- safe visibility frontier

T1, T2, and T3 may all be locally committed while still remaining ahead of
the visibility frontier.

This creates a batching window.  

## Epoch Advance

Eventually, the system advances to a new logical epoch.

Suppose the new visibility frontier becomes:

    6000

Now:

    T1 = 5001
    T2 = 5002
    T3 = 5003

all satisfy:

    transaction_version <= SafeVisibilityFrontier

Therefore, the transactions can become visible together as a batch.

Conceptually:

    Before epoch advance:

        visibility frontier = 5000

        5000 | 5001 5002 5003
             |   T1   T2   T3


    After epoch advance:

        visibility frontier = 6000

        5001 5002 5003 ........ 6000
         T1   T2   T3             |
                                  |
                           new visibility frontier

The epoch transition therefore acts as a batch visibility event.  

## Why Batch Visibility?

If the system advanced visibility independently for every transaction:

    T1 commits -> advance visibility
    T2 commits -> advance visibility
    T3 commits -> advance visibility

it could repeatedly pay synchronization, coordination, and wake-up costs.

Instead, multiple transactions can accumulate and become visible together:

    T1
    T2
    T3
     |
     v
    one epoch advance
     |
     v
    all become visible

This amortizes synchronization overhead across many transactions.

The tradeoff is:

    better batching / throughput efficiency

versus:

    additional waiting latency for individual transactions

This is a classic throughput-vs-latency tradeoff.  

## Visibility-Wait Latency

Transactions that commit early in an epoch generally wait longer for the
next visibility advance than transactions that commit near the epoch
boundary.

Example:

    Epoch N
    |------------------------------------------|
     T1                              T3       advance
      |                               |          |
      |<------- longer wait --------->|          |
                                      |<-short->|

T1 experiences more visibility-wait latency because it committed earlier.

Under a simplified fixed-period model, if transactions arrive uniformly
throughout an epoch of duration:

    T_epoch

then the average visibility wait is approximately:

    E[Wait] ~= T_epoch / 2

Transactions arriving immediately after an epoch boundary may wait close to:

    T_epoch

while transactions arriving just before the next boundary may wait almost
nothing.

This is a useful performance intuition rather than a universal implementation
rule.  

## Latency Decomposition

A transaction's end-to-end latency can include several distinct components:

    T_total
        =
    T_version_assignment
        +
    T_local_commit
        +
    T_visibility_wait
        +
    T_release

For example:

    local commit          = 1 ms
    visibility wait       = 4.5 ms
    release overhead      = 0.1 ms

The storage or durability work may therefore represent only a small portion
of total latency.

This is why performance analysis should decompose the request into logical phases rather than treating all non-compute time as generic overhead.  

## Performance Engineering Questions

Useful questions include:

- How long do transactions spend waiting after local commit?
- How frequently does the visibility frontier advance?
- How many transactions are released per visibility event?
- How much batching benefit is gained?
- How much additional latency does batching introduce?
- Is end-to-end latency dominated by local commit work or visibility waiting?
- How does epoch duration affect average and tail latency?  

## Stage 4 Key Invariants

1. A transaction can be locally committed but not yet visible.

2. Logical version assignment, durability, visibility, and release are
   distinct stages.

3. Multiple committed transactions can accumulate behind a fixed visibility
   frontier.

4. A later epoch advance can move the visibility frontier and make the batch
   visible together.

5. The batching mechanism amortizes synchronization overhead.

6. Early transactions in an epoch typically wait longer than transactions
   that commit near the epoch boundary.

7. The design trades throughput efficiency against additional visibility-wait
   latency.

8. For performance analysis, always separate:

       local work

   from:

       coordination / visibility waiting  

# Stage 5 — Distributed Epoch Coordination  
## Core Idea

In a multi-node distributed system, advancing a local visibility frontier is
not enough to establish that the entire cluster has advanced.

A node may know:

    "I have reached Epoch N+1"

without knowing:

    "Every other node has also reached Epoch N+1"

Therefore, distributed visibility requires a coordination protocol that lets
nodes reason about cluster-wide progress.

## Coordinator-Based Epoch Advancement

A common model is:

    Epoch Coordinator
       |
       +----> Node A
       +----> Node B
       +----> Node C

The coordinator broadcasts an epoch:

    E101

Each node processes the epoch and acknowledges it:

    Node A -> ACK(E101)
    Node B -> ACK(E101)
    Node C -> ACK(E101)

The coordinator does not advance to the next epoch until all required
participants have acknowledged the current one.

Conceptually:

    broadcast E101
        |
        v
    wait for all ACK(E101)
        |
        v
    broadcast E102

This creates a distributed synchronization barrier.

---

## Local Knowledge vs Global Knowledge

Suppose:

    Node A has received E101
    Node B has received E101
    Node C is still at E100

Node A knows:

    "I have reached E101"

but Node A cannot yet conclude:

    "The entire cluster has reached E101"

Therefore:

    Local progress != Knowledge of global progress

This distinction is fundamental in distributed systems.

---

## Why a Later Epoch Proves Something About the Previous One

Assume the protocol rule is:

    The coordinator may issue E102
    only after all nodes acknowledge E101.

Then if Node A later receives:

    E102

Node A can infer:

    Every required node previously acknowledged E101.

Therefore:

    receive(E102)
        =>
    all nodes have reached at least E101

More generally:

    receive(E[N+1])
        =>
    all participants acknowledged E[N]

The important point is that the meaning of the later epoch does not come
from the number itself.

It comes from the protocol rule governing when that epoch may be issued.

Therefore:

    Later epoch
        =
    evidence about global completion of the previous epoch

---

## Epoch Advancement as a Distributed Barrier

The protocol behaves similarly to a barrier synchronization primitive.

Conceptually:

    Node A ACK ----\
    Node B ACK ----- > coordinator may advance
    Node C ACK ----/

The cluster cannot cross the barrier until all required participants have
arrived.

This means an epoch is not merely a monotonically increasing counter.

It represents a distributed progress point.

A better mental model is:

    Epoch Coordinator
        !=
    simple counter generator

Instead:

    Epoch Coordinator
        =
    distributed progress-barrier coordinator

---

## Slowest-Participant Effect

Suppose acknowledgement latency is:

    Node A = 1 ms
    Node B = 2 ms
    Node C = 20 ms

If the coordinator requires acknowledgements from all three nodes, then the
barrier cannot complete before Node C responds.

Conceptually:

    T_barrier ~= max(T_A, T_B, T_C)

So:

    T_barrier ~= 20 ms

This is an important distributed-systems performance property:

    global progress can be determined by the slowest participant

rather than by average node latency.

---

## Sources of Epoch Delay

A node may delay the distributed barrier because of:

- network latency
- CPU saturation
- runtime pauses
- storage stalls
- scheduler delays
- overload
- temporary node-health problems

A single delayed participant can therefore increase cluster-wide progress
latency.

This is especially important for tail-latency analysis.

---

## Knowledge Hierarchy

It is useful to distinguish three levels of knowledge.

### Level 1 — Local State

    Node A has processed E101.

This says nothing by itself about Node B or Node C.

### Level 2 — Coordinator State

    The coordinator has received ACK(E101)
    from all required participants.

The coordinator now knows the cluster has crossed E101.

### Level 3 — Derived Distributed Knowledge

Once Node A receives E102:

    Node A can infer that all required nodes reached E101.

The later coordinator message allows Node A to learn indirectly about
cluster-wide progress.

---

## Connection to Visibility

Suppose Node A has locally completed some transaction work.

Node A may know:

    local commit complete
    local visibility advanced

but still need evidence that other nodes have advanced sufficiently before
making a strong cross-node guarantee.

The epoch barrier provides a mechanism for establishing that evidence.

This is the bridge between:

    local completion

and:

    distributed safety

---

## Performance Engineering View

When epoch advancement stalls, the useful question is not simply:

    "Why didn't the epoch increment?"

Instead ask:

    "Which participant is preventing the distributed barrier from completing?"

Useful metrics include:

- per-node epoch lag
- per-node acknowledgement latency
- time waiting for all acknowledgements
- slowest-node contribution
- epoch advancement interval
- number and duration of epoch stalls

A useful conceptual model is:

    T_epoch_advance
        ~=
    max(per-node acknowledgement latency)
        +
    coordinator overhead

The `max()` term is often the most important one.

---

## Stage 5 Key Invariants

1. A node reaching an epoch does not imply that every other node has reached
   the same epoch.

2. Local progress and knowledge of global progress are different concepts.

3. A coordinator can require acknowledgements from all participants before
   advancing the epoch.

4. Under such a protocol:

       receive(E[N+1])
           =>
       all required participants acknowledged E[N]

5. The later epoch therefore acts as evidence about completion of the
   previous epoch.

6. Epoch advancement can be viewed as a repeated distributed barrier.

7. Barrier latency is often determined by the slowest participant.

8. Distributed coordination can therefore become a major contributor to
   tail latency and request critical paths. 

# Stage 6 — Why Writers Wait Two Epoch Advances but Readers Wait One

## Core Idea

The key distinction is:

    Being at epoch E_N
        !=
    Being able to see commits that occurred during E_N

Commits made during epoch E_N become visible only when the system advances
to E_N+1.

This is because the visibility frontier remains pinned at the start of E_N
while transactions accumulate inside that epoch.

Conceptually:

    E_N opens
        |
        +-- T1 commits
        +-- T2 commits
        +-- T3 commits
        |
        |  commits are collected inside E_N
        |
    E_N+1 opens
        |
        +-- commits from E_N become visible

So:

    E_N      = collection window
    E_N+1    = publication point for E_N's commits

This invariant is what creates the difference between reader and writer
wait semantics.

---

## Reader Case

Suppose a reader observes a snapshot at:

    E_N

The reader wants to ensure that a later read routed to another node cannot
return a state older than E_N.

Therefore the reader needs proof that:

    all nodes have reached at least E_N

Under the epoch coordination protocol:

    receive E_N+1
        =>
    all nodes acknowledged E_N

Therefore:

    Reader at E_N
        |
        v
    needs all nodes >= E_N
        |
        v
    E_N+1 provides that proof
        |
        v
    safe to return

So the reader requires:

    1 epoch advance

Conceptually:

    Reader snapshot = E_N
    Proof arrives   = E_N+1

---

## Writer Case

Now suppose a transaction commits during:

    E_N

The write is different from the reader because the transaction's commit is
not visible merely when nodes are at E_N.

The transaction was created inside E_N.

It becomes visible only when the visibility frontier advances to:

    E_N+1

Therefore, for the writer to be globally safe, it needs:

    all nodes to reach E_N+1

Only then can every node see the commits that occurred during E_N.

But how does Node A know that all nodes reached E_N+1?

Using the same epoch coordination invariant:

    receive E_N+2
        =>
    all nodes acknowledged E_N+1

Therefore:

    Writer commits during E_N
        |
        v
    commit becomes visible at E_N+1
        |
        v
    need all nodes to reach E_N+1
        |
        v
    E_N+2 provides proof
        |
        v
    safe to release / acknowledge

So the writer requires:

    2 epoch advances

Conceptually:

    Commit epoch    = E_N
    Visibility      = E_N+1
    Global proof    = E_N+2

---

## Reader vs Writer

The asymmetry can be summarized as:

    Reader at snapshot E_N:
        needs all nodes >= E_N
        proof arrives with E_N+1
        => 1 advance

    Writer committed during E_N:
        needs all nodes >= E_N+1
        because E_N commits become visible at E_N+1
        proof arrives with E_N+2
        => 2 advances

The writer is one step further out because it creates new state inside the
current epoch.

The reader is validating an already-existing visibility point.

---

## Why E_N Commits Are Not Visible During E_N

During an epoch, the visibility frontier remains at the epoch boundary while
transactions receive newer logical versions inside the epoch.

Example:

    visibility frontier = 5000

    T1 = 5001
    T2 = 5002
    T3 = 5003

Conceptually:

    5000 | 5001 5002 5003
         |   T1   T2   T3
         |
         +--- visibility frontier

T1, T2, and T3 are ahead of the visibility frontier.

When the next epoch opens:

    visibility frontier -> 6000

then:

    5001, 5002, 5003 <= 6000

and all prior-epoch commits become visible together.

Therefore:

    epoch E_N
        =
    batching / collection window

and:

    transition to E_N+1
        =
    publication event for E_N's commits

---

## Distributed Proof Rule

The distributed coordination rule is:

    coordinator sends E_N+1
    only after all required nodes acknowledge E_N

Therefore:

    receive(E_N+1)
        =>
    all required nodes reached E_N

This lets nodes infer global progress indirectly.

Applying it twice gives the writer rule:

    receive(E_N+2)
        =>
    all nodes reached E_N+1
        =>
    all nodes can see commits made during E_N

---

## Concrete Example

Suppose transaction T1 commits during:

    E100

Then:

    E100
      |
      +-- T1 commits
      |
      v
    E101

At E101:

    T1 becomes visible locally as the visibility frontier advances.

But Node A still needs proof that every node reached E101.

That proof arrives when:

    E102

is issued.

So:

    T1 commits in E100
        |
        v
    E101 makes E100 commits visible
        |
        v
    all nodes ACK E101
        |
        v
    E102 is issued
        |
        v
    Node A knows every node can see T1
        |
        v
    safe ACK

---

## Why a Reader Needs Less Waiting

Suppose a reader observes snapshot:

    E100

The reader does not need every node to see commits made during E100.

It only needs every node to have reached:

    E100

That proof arrives when:

    E101

is issued.

So:

    reader at E100
        |
        v
    E101 proves all nodes reached E100
        |
        v
    safe return

This is why:

    reader = 1 epoch advance

while:

    writer = 2 epoch advances

---

## Performance Engineering View

The two-epoch writer wait is not arbitrary overhead.

It reflects two logical requirements:

    1. Move the visibility frontier far enough to include the write.
    2. Prove that this new visibility point propagated across the cluster.

Therefore writer latency can include:

    local commit work
        +
    wait for publication boundary
        +
    wait for distributed propagation proof

A useful conceptual model is:

    T_write
        ~=
    T_local_commit
        +
    T_visibility_wait
        +
    T_global_propagation_wait

The final component may be dominated by the slowest participant in the
cluster.

This means a write-latency spike can be caused by another node's delay,
even when the writer's local storage path is healthy.

---

## Stage 6 Key Invariants

1. Commits made during E_N are not visible during E_N.

2. E_N+1 acts as the publication point for commits made inside E_N.

3. A reader at E_N only needs proof that every node reached E_N.

4. That proof arrives with E_N+1.

5. Therefore a reader waits 1 epoch advance.

6. A writer committing during E_N needs every node to reach E_N+1,
   because that is when E_N's commits become visible.

7. Proof that every node reached E_N+1 arrives with E_N+2.

8. Therefore a writer waits 2 epoch advances.

9. The asymmetry exists because:

       Reader:
           validates an existing visibility point

       Writer:
           creates new state that must first become visible,
           then globally proven visible

10. From a performance perspective, writer latency may include both
    publication wait and distributed-propagation wait.  



