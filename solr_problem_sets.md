## Stage 8 — Programming Capstone and Experimental Validation

The goal of this stage is to experimentally validate the search cost models developed in Stages 1–7 using a small, transparent Python implementation.

The objective is **not to recreate Lucene**. The purpose is to make the underlying algorithmic work visible and measurable:

- what is traversed
- what work grows with vocabulary size
- what work grows with postings size
- how query shape changes term-discovery cost
- how n-grams move cost from query time to indexing time

---

### 1. Minimal Inverted Index

Start with the simplest useful representation:

```text
term → sorted DocIDs
```

Example:

```text
sameer → [1, 2, 3]
khan   → [1, 4]
ahmed  → [2]
john   → [3]
samir  → [4]
```

A minimal Python representation is:

```python
dict[str, list[int]]
```

If documents receive monotonically increasing DocIDs, postings can remain sorted through simple append operations:

```text
increasing DocIDs
      +
append-only postings
      ↓
sorted postings
```

This avoids arbitrary insertion into the middle of a postings list.

A richer implementation could later replace each DocID with a posting containing:

```text
doc_id
term_frequency
positions
offsets
```

but the minimal implementation is sufficient for understanding the search algorithms.

---

### 2. Boolean Postings Operations

Sorted postings enable efficient Boolean operations using advancing cursors.

For example:

```text
sameer → [1, 3, 4, 6]
khan   → [1, 2, 5, 6]
```

For:

```text
sameer AND khan
```

two cursors advance through the sorted lists:

```text
[1, 3, 4, 6]
 ↑

[1, 2, 5, 6]
 ↑
```

When the DocIDs match, the document is emitted.

The result is:

```text
[1, 6]
```

For postings lists of lengths `m` and `n`, simple merge-style intersection has complexity:

```text
O(m + n)
```

Important metrics:

```text
cursor comparisons
postings processed
```

This makes the value of sorted postings directly observable.

---

### 3. Exact Search

Exact search performs a direct dictionary lookup followed by postings traversal:

```text
sameer
   ↓
dictionary lookup
   ↓
postings
```

Cost model:

```text
T_exact
≈
T_termLookup
+
T_postings
```

The important property is that exact lookup does not require examining unrelated vocabulary terms.

Increasing the vocabulary from:

```text
10K
```

to:

```text
100K
```

to:

```text
1M
```

does not turn exact lookup into a vocabulary scan.

The dominant work after term discovery depends on the size of the term's postings list.

---

### 4. Sorted Term Dictionary

Prefix search requires ordered term space.

Maintain two structures:

```text
postings:
    term → sorted DocIDs

sorted_terms:
    lexicographically sorted terms
```

For example:

```text
ahmed
john
khan
sameer
sameera
sameerabc
samir
```

The postings map provides exact lookup.

The sorted dictionary provides ordered navigation through term space.

This is deliberately simpler than Lucene's real term dictionary/FST representation.

---

### 5. Prefix Search

Consider:

```text
sameer*
```

Because the dictionary is lexicographically ordered, terms sharing the prefix `sameer` occupy a contiguous region.

The algorithm becomes:

```text
seek("sameer")
      ↓
first term >= "sameer"
      ↓
enumerate forward
      ↓
process matching terms
      ↓
stop when prefix no longer matches
```

Cost model:

```text
T_prefix
≈
T_seek
+
T_termEnumeration
+
T_postings
```

A useful experiment is:

```text
s*
sa*
sam*
same*
sameer*
```

As the prefix becomes more selective:

```text
matching terms ↓
term enumeration ↓
postings work ↓
```

The key advantage is that unrelated portions of the vocabulary do not need to be enumerated.

---

### 6. Leading Wildcard

Consider:

```text
*sameer
```

The suffix constraint does not align with the forward lexicographic ordering of the term dictionary.

In the simplified implementation, the algorithm can deliberately examine dictionary terms and test:

```python
term.endswith("sameer")
```

Conceptually:

```text
dictionary exploration
      ↓
suffix test
      ↓
matching terms
      ↓
postings
```

Cost model:

```text
T_leadingWildcard
≈
T_dictionaryExploration
+
T_termExpansion
+
T_postings
```

A useful controlled experiment is to compare:

```text
sameer*
```

with:

```text
*sameer
```

while keeping approximately constant:

```text
M = matching terms
P = postings volume
```

If the leading wildcard examines substantially more dictionary terms, it demonstrates:

```text
same result count
≠
same query cost
```

The difference is term-discovery work.

---

### 7. Arbitrary Substring Wildcard

Consider:

```text
*sameer*
```

There is:

```text
no fixed prefix
no fixed suffix boundary
substring may occur anywhere
```

A deliberately naive implementation examines each term and tests:

```python
"sameer" in term
```

Conceptually:

```text
broad term-space exploration
      ↓
substring test
      ↓
matching terms
      ↓
postings
```

Cost model:

```text
T_substring
≈
T_broadDictionaryExploration
+
T_termExpansion
+
T_postings
```

One of the most important controlled experiments is to increase vocabulary size while holding actual matches approximately constant:

```text
V = 10K    → ~10 matches
V = 100K   → ~10 matches
V = 1M     → ~10 matches
```

The output size remains roughly constant while dictionary exploration grows.

This experimentally demonstrates:

```text
small result set
≠
cheap query
```

---

### 8. N-Gram Indexing

Substring search can instead move work from query time to indexing time.

For trigrams:

```text
sameer
  ↓
sam
ame
mee
eer
```

For a token of length `L` and gram size `n`:

```text
G = L - n + 1
```

Build a separate inverted index:

```text
gram → sorted DocIDs
```

For example:

```text
sam → [...]
ame → [...]
mee → [...]
eer → [...]
```

Each gram has now become an ordinary indexed term.

Searching for:

```text
sameer
```

can therefore begin with:

```text
exact lookup("sam")
exact lookup("ame")
exact lookup("mee")
exact lookup("eer")
```

followed by postings intersection:

```text
P(sam)
  ∩
P(ame)
  ∩
P(mee)
  ∩
P(eer)
      ↓
candidate documents
```

Cost model:

```text
T_ngram
≈
Σ T_exactLookup(gram_i)
+
T_postingsIntersection
+
T_verification
```

The postings lists can be processed from shortest to longest so that selective grams reduce the intermediate candidate set early.

---

### 9. Candidate Retrieval vs Verification

Gram intersection does not necessarily prove that the original substring occurs contiguously.

A document could contain:

```text
sam ... ame ... mee ... eer
```

All four grams are present, so the document may survive the postings intersection.

However, it does not necessarily contain:

```text
sameer
```

Therefore:

```text
gram intersection
      ↓
candidate documents
      ↓
verification
      ↓
final results
```

This means:

```text
candidate count C
≠
final result count
```

Verification can use:

```text
Option A:
original stored text

Option B:
positions / offsets
```

A useful experiment deliberately creates false-positive documents and measures:

```text
candidate count
actual match count
false-positive count
false-positive rate
```

N-gram retrieval should therefore be understood as candidate generation followed by exact verification.

---

### 10. N-Gram Index Amplification

N-grams reduce query-time term discovery by doing additional work during indexing.

For a token of length `L` and gram size `n`:

```text
G = L - n + 1
```

For example:

```text
sameer
```

has length:

```text
L = 6
```

For trigrams:

```text
G = 6 - 3 + 1
  = 4
```

producing:

```text
sam
ame
mee
eer
```

Useful index measurements include:

```text
original term occurrences
generated gram occurrences
unique grams
total gram postings
```

This exposes the storage and write amplification introduced by n-gram indexing.

---

### 11. Gram-Size Tradeoff

Compare:

```text
n = 2
n = 3
n = 4
n = 5
```

Measure:

```text
index size
unique grams
average postings-list length
maximum postings-list length
candidate count
verification count
query latency
```

Smaller `n` generally means:

```text
more grams
less selective grams
larger postings
larger candidate sets
better short-query support
```

Larger `n` generally means:

```text
fewer grams
more selective grams
smaller candidate sets
weaker short-query support
```

Therefore gram size represents a tradeoff between:

```text
index amplification
query selectivity
candidate volume
short-query support
```

---

### 12. Unified Cost Model

The experiments can be interpreted using:

```text
T_query
=
T_termDiscovery
+
T_postings
+
T_setOperations
+
T_verification
```

Important variables:

```text
V = number of unique dictionary terms
M = number of matching terms
P = postings volume
G = number of query grams
C = candidate documents
```

Different query types move work between these components.

#### Exact

```text
T_termDiscovery
≈ direct lookup

T_postings
≈ postings for one term
```

#### Prefix

```text
T_termDiscovery
≈ seek + bounded term enumeration
```

#### Leading Wildcard

```text
T_termDiscovery
≈ broader dictionary exploration
```

#### Arbitrary Substring Wildcard

```text
T_termDiscovery
≈ broad term-space exploration
```

#### N-Gram

```text
T_termDiscovery
≈ G exact gram lookups

T_setOperations
≈ postings intersections

T_verification
≈ candidate verification
```

---

### 13. Instrumentation

For every query, collect algorithmic work counters such as:

```text
dictionary lookups
terms examined
matching terms
postings processed
intersection comparisons
gram lookups
candidate count
verification count
final result count
wall-clock latency
```

Wall-clock latency is useful, but the algorithmic counters are more important for understanding why performance changes.

Runtime measurements can contain noise.

Work counters directly expose how the algorithm scales.

---

### 14. Controlled Experiment — Vocabulary Scaling

Hold final result count approximately constant while increasing vocabulary size:

```text
V = 10K    → ~10 matches
V = 100K   → ~10 matches
V = 1M     → ~10 matches
```

Compare:

```text
exact
prefix
substring wildcard
```

Expected behavior:

```text
Exact lookup:
relatively insensitive to unrelated V

Selective prefix:
skips most unrelated vocabulary

Substring wildcard:
terms examined grows substantially with V
```

This demonstrates:

```text
result count
≠
query cost
```

---

### 15. Controlled Experiment — Prefix Selectivity

Keep the corpus fixed and execute:

```text
s*
sa*
sam*
same*
sameer*
```

Measure:

```text
M
dictionary terms enumerated
P
latency
```

Expected behavior:

```text
longer / more selective prefix
        ↓
fewer matching terms
        ↓
less term enumeration
        ↓
less postings work
```

---

### 16. Controlled Experiment — Wildcard vs N-Gram

Compare:

```text
*sameer*
```

against trigram search for:

```text
sameer
```

For wildcard search measure:

```text
terms examined
matching terms
postings processed
```

For n-gram search measure:

```text
gram lookups
df of each gram
intersection comparisons
candidate count
verification count
```

Also measure indexing cost and index size.

The expected architectural tradeoff is:

```text
Wildcard
    ↓
less specialized indexing
    ↓
more query-time term discovery
```

versus:

```text
N-Gram
    ↓
index-time precomputation
+
storage amplification
    ↓
exact gram lookups
+
postings intersection
+
verification
```

N-grams therefore shift cost from:

```text
query-time term discovery
```

toward:

```text
index-time precomputation + storage
```

---

### 17. Immutable Segments

The initial implementation uses one mutable structure:

```text
term → postings
```

A useful extension is to model immutable segments:

```text
Segment 1
Segment 2
Segment 3
```

Each segment contains:

```text
its own term dictionary
its own postings lists
local DocIDs
```

New documents create new segments instead of requiring arbitrary insertion into old sorted postings lists.

Conceptually:

```text
existing immutable segments
          +
new immutable segment
```

Queries search each segment:

```text
query
  ↓
Segment 1
Segment 2
Segment 3
  ↓
merge results
```

Segments can later be merged:

```text
Segment 1 ┐
Segment 2 ├──→ larger merged segment
Segment 3 ┘
```

This explains why immutable sorted structures avoid arbitrary in-place insertion into old postings lists and connects the simplified implementation to Lucene's segment architecture.

---

### 18. Distributed Search Simulation

The same local search mechanics can be repeated across independent shards:

```text
              Coordinator
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Shard 1     Shard 2     Shard 3
```

Each shard performs its own:

```text
term discovery
postings processing
set operations
verification
```

Total cluster work is approximately:

```text
W_cluster
≈
Σ W_shard
```

But user-visible latency is closer to:

```text
T_user
≈
max(T_shard)
+
T_coordination
```

For example:

```text
Shard 1 = 20 ms
Shard 2 = 23 ms
Shard 3 = 21 ms
Shard 4 = 150 ms
```

Total shard work is roughly:

```text
20 + 23 + 21 + 150
=
214 ms
```

but user-visible search latency is closer to:

```text
150 ms
+
coordination
```

Therefore:

```text
total cluster work
≠
user-visible latency
```

A deliberately slow shard can also be introduced to demonstrate tail-latency effects.

---

### 19. Simplified Implementation vs Real Lucene

The capstone deliberately uses transparent structures such as:

```text
Python dict
Python list
sorted term arrays
binary search
explicit cursor intersections
naive wildcard scans
simple tokenization
raw DocID postings
```

It does not attempt to reproduce Lucene's production implementation details.

The design principle is:

```text
Do not optimize away
the behavior being studied.
```

First expose:

```text
dictionary traversal
postings traversal
comparisons
term expansion
candidate generation
verification
```

Then optimize only after the algorithmic behavior is understood.

---

### 20. Final Mental Model

The entire search pipeline can be viewed as:

```text
                 QUERY
                   │
                   ↓
          ┌──────────────────┐
          │ TERM DISCOVERY   │
          └──────────────────┘
                   │
                   ↓
          ┌──────────────────┐
          │ POSTINGS ACCESS  │
          └──────────────────┘
                   │
                   ↓
          ┌──────────────────┐
          │ SET OPERATIONS   │
          └──────────────────┘
                   │
                   ↓
          ┌──────────────────┐
          │ VERIFICATION     │
          └──────────────────┘
                   │
                   ↓
                RESULTS
```

Different query shapes stress different portions of this pipeline.

```text
Exact
  ↓
direct term discovery
  ↓
postings


Prefix
  ↓
seek
+
bounded enumeration
  ↓
postings


*sameer
  ↓
broader dictionary exploration
  ↓
postings


*sameer*
  ↓
broad term-space exploration
  ↓
postings


N-Gram Substring
  ↓
exact gram lookups
  ↓
postings intersections
  ↓
candidate documents
  ↓
verification
```

The fundamental relationship is:

```text
Query Shape
     ↓
Indexed Representation
     ↓
Term Discovery Algorithm
     ↓
Postings / Set-Operation Work
     ↓
Verification
     ↓
Latency
```

The key performance-engineering questions are therefore:

```text
How are matching terms discovered?

How much dictionary space is explored?

How many terms are expanded?

How much postings data is processed?

What set operations are required?

How many candidates require verification?

What work was moved to indexing time?

How is this work repeated across shards?
```

The central principles demonstrated by the capstone are:

```text
result count ≠ query cost

matching-term count ≠ term-discovery cost

term selectivity ≠ document selectivity

candidate count ≠ final result count

total cluster work ≠ user-visible latency
```

Most importantly:

> **Search performance depends on how well the query shape aligns with the indexed representation.**

A query is not inherently cheap or expensive solely because it returns few or many results. Its cost depends on the work required to discover matching terms, traverse postings, perform set operations, verify candidates, and repeat that work across the execution topology.  

