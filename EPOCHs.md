# Epochs in Distributed Systems  
## Stage 1 - Why do we need Epochs?  
### The Core Problem  
In a distributed database, completing a transaction locally does NOT mean the transaction is globally visible.  

For example:  
 Initial State  
 Node A visibility: 100  
 Node B visibility: 100  
 Node C visibility: 100  

A client writes version 101 on Node A. After the local commit completes, the system may temporarily be in the following state:  
 Node A visibility: 101  
 Node B visibility: 101  
 Node C visibility: 100  

Therefore,  
 **Local commit != Global visibility**  

If Node A ACKs the write at this point, a subsequent read could be routed to Node C and return version 100.  
This would violate **cross-node read-after-write consistency guarantee**.  

### Cross-Node Read-After-Write  
If a write has completed before a subsequent read begins:  
````
    WRITE(101) completes
             |
             v
            ACK
             |
             v
       READ on Node B/C
````
then the later read must not observe a version older than 101.  

Conceptually:

    WRITE(V) completed
        =>
    subsequent READ >= V

Therefore, a write ACK may need to represent more than local durability.

It may need to mean:

    "The distributed system has advanced sufficiently that a subsequent
     operation routed to another node will not fall behind this write."

### Overlapping Operations Are Different

Consider:

    WRITE(101)
    |---------------------|
             READ
             |------|

The read overlaps the write.

If the read returns version 100, this is not necessarily a consistency
violation, because the write had not yet completed when the read started.

There is no happens-before relationship:

    WRITE_complete < READ_start

Contrast this with:

    WRITE(101)
    |---------| ACK

                  READ
                  |-----|

Now:

    WRITE_complete < READ_start

and a cross-node read-after-write guarantee requires the read to see
101 or something newer.

### Monotonic Reads  
A related problem occurs between two reads.

If:

    READ #1 -> version 101

then a later read routed to another node should not return:

    READ #2 -> version 100

Conceptually:

    READ(V)
       =>
    later READ >= V

This is a monotonic-read property.

### Distributed Safe Visibility Frontier  
Suppose nodes have progressed to:

    Node A = 108
    Node B = 105
    Node C = 107

The highest version known to be safe across the entire cluster is:

    safe_frontier = min(108, 105, 107)
                  = 105

Therefore:

    <= 105 : known globally safe
    >  105 : may exist only on a subset of nodes

General model:

    SafeVisibilityFrontier =
        min(per-node visibility frontiers)

This introduces an important distributed-systems distinction:

    Something being true on one node
        !=
    that node knowing it is globally true.

A distributed coordination mechanism is needed to establish proof that
all relevant nodes have crossed a particular visibility point.  

### Performance Engineering View

Suppose:

    local commit time       = 1 ms
    total write latency     = 15 ms

The remaining 14 ms may not be "miscellaneous overhead."

It may be:

    local work
        |
        v
    distributed coordination / visibility barrier
        |
        v
    safe ACK

Optimizing local I/O from 1 ms -> 0.5 ms would barely improve the
15 ms request if distributed coordination dominates the critical path.

Therefore, when analyzing latency, ask:

    What correctness condition is this request waiting for?

    Is the request waiting for:
      - computation?
      - I/O?
      - durability?
      - distributed propagation?
      - a synchronization barrier?  

### Slowest-Participant Effect

If cluster-wide progress requires every node to cross a frontier:

    A = 1 ms
    B = 2 ms
    C = 15 ms

then barrier completion is approximately determined by:

    max(1, 2, 15) = 15 ms

So distributed coordination latency is often dominated by the slowest
participant rather than the average participant.

This makes node stalls, network delays, GC pauses, overload, etc. potential
contributors to cluster-wide tail latency.  

## Stage 1 Key Invariants

    1. Local commit != global visibility.

    2. Requests may move between nodes with different visibility frontiers.

    3. Once a write is ACKed, a later cross-node read must not go backward
       if the system promises read-after-write consistency.

    4. The writer therefore needs evidence of sufficiently global progress
       before safely returning the ACK.

    5. A useful abstraction is a distributed safe visibility frontier.

    6. Global progress is bounded by the slowest participating node.

    7. Distributed coordination can therefore sit directly on the
       request's latency critical path.  

