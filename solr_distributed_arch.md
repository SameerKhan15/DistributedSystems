## Stage 7 — Distributed Solr Execution

SolrCloud extends local Lucene search by distributing a collection across **shards**. The local Lucene mechanics remain the same; they now execute independently on each relevant shard and their results are combined by a coordinator.

### Architecture

```text
Collection
   ↓
Shard
   ↓
Replica
   ↓
Lucene Index
   ↓
Segments
   ↓
FST / BlockTree / Postings
```

- **Collection** — logical dataset being searched.
- **Shard** — partition containing a subset of the collection.
- **Replica** — copy of a shard used for availability, fault tolerance, and read capacity.
- **Segment** — immutable Lucene index structure inside a replica.

Shards **partition** the data; replicas **duplicate** those partitions. A distributed query normally executes against one selected replica from each relevant shard.

### Distributed Query Execution

```text
                 Client
                    |
                    v
               Coordinator
              /     |     \
             v      v      v
          Shard 1 Shard 2 Shard 3
             |      |      |
             v      v      v
           Lucene Lucene  Lucene
             |      |      |
             v      v      v
           Local Top-K results
              \     |     /
               \    |    /
                v   v   v
               Coordinator
                    |
                    v
               Global Top-K
```

Each shard independently performs the local Lucene work:

```text
term discovery
    ↓
postings
    ↓
set operations
    ↓
scoring
    ↓
local Top-K
```

The coordinator merges the local candidates:

```text
TopK_global
=
TopK(
    TopK(S1)
    ∪ TopK(S2)
    ∪ ...
    ∪ TopK(Sn)
)
```

### Distributed Cost Model

For shard `i`:

```text
T_i
≈
T_termDiscovery,i
+
T_postings,i
+
T_setOps,i
+
T_scoring,i
+
T_localTopK,i
```

Distributed latency is approximately:

```text
T_query
≈
T_fanout
+
max_i(T_i)
+
T_network
+
T_globalMerge
+
T_optionalFetch
```

Because shards execute largely in parallel, latency is usually controlled by the **slowest required shard**, not the sum of shard execution times.

Total cluster work behaves differently:

```text
W_global
≈
Σ W_i
```

Therefore:

```text
user-visible latency → max-like behavior
total cluster work   → sum-like behavior
```

This distinction is fundamental for distributed-search performance analysis.

### Stragglers and Tail Latency

For example:

```text
Shard 1 → 40 ms
Shard 2 → 42 ms
Shard 3 → 45 ms
Shard 4 → 190 ms
```

The 190-ms shard can dominate the distributed response:

```text
T_distributed
≈
max(T_shards)
+
T_network
+
T_merge
```

More shards provide greater parallelism, but also increase exposure to slow shard responses.

### Deep Pagination

For:

```text
start = 100000
rows  = 10
```

each shard may need roughly:

```text
start + rows
```

candidate entries.

Across `S` shards:

```text
candidate work
≈
S × (start + rows)
```

Therefore:

```text
final result count ≠ internal candidate work
```

Returning only 10 rows can still require a very large distributed candidate set.

### Two-Phase Retrieval

Distributed retrieval can separate ranking from stored-field retrieval.

**Phase 1:**

```text
all shards
    ↓
candidate DocIDs + scores/sort values
    ↓
coordinator
    ↓
global Top-K
```

**Phase 2:**

```text
global winners
    ↓
relevant shards
    ↓
fetch requested stored fields
```

This trades an additional network round trip for potentially fewer transferred bytes.

### Shard Count Tradeoff

More shards can provide:

- smaller local indexes
- more parallelism
- more aggregate CPU/I/O bandwidth

But they also increase:

- RPC fan-out
- coordinator merge work
- network communication
- tail-latency exposure
- operational complexity

Therefore:

```text
more shards ≠ automatically faster
```

### Routing

Without routing, a query may need to fan out across many shards.

With an appropriate routing key, Solr may reduce:

```text
S_effective = 20
```

to:

```text
S_effective = 1
```

This leads to a useful cluster-level work model:

```text
W_cluster
≈
shardsTouched × localSearchCost
```

There are therefore two independent optimization levers:

```text
1. Reduce localSearchCost
2. Reduce shardsTouched
```

### Local Query Cost Multiplies Across Shards

For an expensive substring wildcard:

```text
*sameer*
```

across 20 shards:

```text
W_cluster
≈
20 × W_wildcard_perShard
```

If an n-gram index transforms substring search into targeted gram lookups:

```text
sameer
   ↓
sam, ame, mee, eer
   ↓
exact gram lookups
   ↓
postings intersection
   ↓
optional verification
```

then:

```text
W_cluster
≈
20 × W_ngram_perShard
```

A local Lucene optimization can therefore become a significant **fleet-level optimization**, because the savings repeat across every participating shard.

### Final Mental Model

```text
CLIENT
   |
   v
SOLR COORDINATOR
   |
   +---------------------------+
   |             |             |
   v             v             v
SHARD 1        SHARD 2       SHARD 3
   |             |             |
select         select        select
replica        replica       replica
   |             |             |
   v             v             v
LUCENE         LUCENE        LUCENE
   |
Segments
   ↓
Term Discovery
   ↓
Postings
   ↓
Set Operations / Scoring
   ↓
Local Top-K
   |
   +-------------+-------------+
                 |
                 v
            Global Top-K
                 |
                 v
       Optional Field Fetch
                 |
                 v
               CLIENT
```

The central systems model is:

```text
Distributed Solr cost
=
local Lucene work
+
distributed coordination
```

For resource consumption:

```text
W_cluster
≈
shardsTouched × localSearchCost
```

For user-visible latency:

```text
T_query
≈
fanout
+
slowest required shard
+
network
+
global merge
+
optional fetch
```

**Core insight:** data structures and query algorithms determine the cost of search inside each Lucene shard; sharding, routing, fan-out, Top-K merging, pagination, network communication, and stragglers determine how that local cost behaves at distributed-system scale.  

