# Solr / Lucene Search from First Principles — Stage 1

## Exact Term Lookup with an Inverted Index

## Goal

Understand, algorithmically, how an exact keyword query such as:

```text
sameer
```

is resolved using an inverted index.

This stage intentionally excludes:

* prefix wildcard queries such as `sameer*`
* leading wildcard queries such as `*sameer`
* substring queries such as `*sameer*`
* n-gram indexing
* Solr shard/replica distribution
* advanced Lucene implementation details

The goal is to first understand the simplest and most important primitive:

> Given one exact indexed term, how does Lucene find the matching documents efficiently?

---

## 1. Start with Documents

Assume the searchable field contains:

```text
D1: "Sameer Khan"
D2: "Ahmed Khan"
D3: "Sameer Ahmed"
D4: "John Sameer"
D5: "Samir Khan"
D6: "Sameer"
```

Before building the index, Lucene runs the field's analyzer.

For this toy example, assume:

```text
tokenize on whitespace
        ↓
lowercase
```

The analyzed representation becomes:

```text
D1 → [sameer, khan]
D2 → [ahmed, khan]
D3 → [sameer, ahmed]
D4 → [john, sameer]
D5 → [samir, khan]
D6 → [sameer]
```

Important:

> Lucene searches indexed terms produced by the analyzer, not necessarily the literal original string.

A production Solr field may have a much richer analyzer chain.

---

## 2. The Naive Search Model

The data naturally starts as:

```text
Document → Terms
```

For example:

```text
D1 → sameer, khan
D2 → ahmed, khan
D3 → sameer, ahmed
D4 → john, sameer
D5 → samir, khan
D6 → sameer
```

If this were the only representation available, an exact search for:

```text
sameer
```

could be implemented as:

```text
for each document:
    inspect its terms
    if "sameer" is present:
        return the document
```

Result:

```text
D1 → match
D2 → no
D3 → match
D4 → match
D5 → no
D6 → match
```

So the answer is:

```text
[D1, D3, D4, D6]
```

This works, but it requires inspecting documents across the corpus.

For a corpus with `N` documents, this is conceptually close to:

```text
O(N)
```

or, more precisely, proportional to the amount of searchable text/tokens scanned.

That is not attractive at search-engine scale.

---

## 3. Invert the Relationship

Instead of relying on:

```text
Document → Terms
```

we build:

```text
Term → Documents
```

For the example above:

```text
ahmed  → [D2, D3]
john   → [D4]
khan   → [D1, D2, D5]
sameer → [D1, D3, D4, D6]
samir  → [D5]
```

This is the fundamental idea behind an **inverted index**.

The list:

```text
sameer → [D1, D3, D4, D6]
```

is the postings list for the term `sameer`.

Each individual document occurrence in that list is a posting.

---

## 4. Why "Inverted"?

The original orientation is:

```text
Document → Terms
```

The inverted orientation is:

```text
Term → Documents
```

This is similar to the index at the back of a book.

A normal book is conceptually:

```text
page → words
```

A book index reverses the relationship:

```text
"consensus"           → pages 81, 84, 120
"distributed systems" → pages 37, 102, 151
```

Lucene's inverted index is a highly optimized computational version of the same idea.

---

## 5. Two Important Structures

For reasoning purposes, separate the index into two conceptual structures.

### 5.1 Term Dictionary

The dictionary contains indexed terms:

```text
ahmed
john
khan
sameer
samir
```

Its job is to answer:

> Does the requested term exist, and where is its postings information?

### 5.2 Postings Lists

The postings structure maps terms to matching document IDs:

```text
ahmed  → [2, 3]
john   → [4]
khan   → [1, 2, 5]
sameer → [1, 3, 4, 6]
samir  → [5]
```

Its job is to answer:

> Which documents contain this term?

So exact-term retrieval can be decomposed into:

```text
find term
   ↓
find documents containing term
```

or:

```text
Term Dictionary Lookup
        ↓
Postings Traversal
```

This separation becomes extremely important when studying wildcard queries later.

---

## 6. Execute the Exact Query `sameer`

Assume the query analyzer also lowercases the user input.

```text
"sameer"
    ↓
query analyzer
    ↓
sameer
```

Lucene then searches the term dictionary:

```text
ahmed
john
khan
sameer   ← found
samir
```

The dictionary entry for `sameer` points to its postings data:

```text
sameer
   │
   ▼
[1, 3, 4, 6]
```

Those are the matching internal document IDs.

The critical observation:

> Lucene did not inspect every document to discover which documents contain `sameer`.

Instead:

```text
sameer
   ↓
term dictionary
   ↓
postings list
   ↓
[1, 3, 4, 6]
```

That direct jump is the central benefit of an inverted index.

---

## 7. A Simple Cost Model

For an exact term query, a useful first approximation is:

```text
T_query
  =
T_term_lookup
  +
T_postings_processing
```

For `sameer`:

```text
T_query
  =
T_find("sameer")
  +
T_read_postings("sameer")
```

Define document frequency:

```text
df(term) = number of documents containing the term
```

For the toy index:

```text
df(sameer) = 4
df(khan)   = 3
df(john)   = 1
```

The postings-processing cost is strongly related to the length of the postings list:

```text
T_postings ∝ df(term)
```

This leads to an important performance insight.

Suppose:

```text
N = 1,000,000,000 documents
```

but a rare term occurs in only:

```text
df(rare_term) = 7
```

The search engine does not need to scan one billion documents to find those seven.

It can:

```text
rare_term
    ↓
term dictionary
    ↓
7-entry postings list
```

So exact-query cost is not simply proportional to total corpus size.

---

## 8. Internal DocIDs vs Full Documents

The postings list normally points to compact internal document identifiers:

```text
sameer → [1, 3, 4, 6]
```

It does not need to contain the entire Account record.

Conceptually:

```text
Term search
    ↓
matching internal DocIDs
    ↓
scoring / ranking
    ↓
retrieve stored fields as required
```

Search engines therefore separate two concerns:

1. efficiently determine which documents match
2. retrieve/display document contents when needed

---

## 9. Postings Can Store More Than DocIDs

The simplified model is:

```text
sameer → [1, 3, 4, 6]
```

But Lucene may store richer information depending on the field configuration, such as:

```text
DocID
term frequency
positions
offsets
```

Example:

```text
D1 = "sameer khan sameer"
```

For the term `sameer`, the index might conceptually contain:

```text
DocID:      1
frequency:  2
positions:  [0, 2]
```

Positions are useful for phrase queries such as:

```text
"sameer khan"
```

because Lucene can reason about adjacency using positions instead of re-reading every original string.

For Stage 1, however, the important abstraction remains:

```text
term → sorted DocIDs
```

---

## 10. Postings can contain richer metadata

A posting does not have to contain only:

```text id="6o5byc"
term → DocID
```

It can also record information about **how the term occurs inside the document**.

Conceptually, a posting may contain:

```text id="shxwkw"
DocID
term frequency
positions
offsets
```

Consider:

```text id="g5m8w4"
D1 = "sameer khan sameer"
```

After tokenization:

```text id="8r0pzc"
position 0 → sameer
position 1 → khan
position 2 → sameer
```

The posting for `sameer` could therefore be represented conceptually as:

```text id="jdfowb"
sameer:
  DocID      = 1
  frequency  = 2
  positions  = [0, 2]
```

#### Term frequency

```text id="pxh7x6"
frequency = 2
```

means that `sameer` occurs twice in this document.

This information can participate in scoring and other query processing.

---

#### Positions

A **position** identifies where a token appears in the analyzed token stream.

For:

```text id="2nlyc7"
sameer khan sameer
```

the token stream is:

```text id="9wzkwj"
0: sameer
1: khan
2: sameer
```

Therefore:

```text id="9g94ka"
sameer → positions [0, 2]
khan   → position  [1]
```

Positions allow the search engine to reason about relationships between terms.

For example, consider the phrase query:

```text id="enrxmw"
"sameer khan"
```

The engine can compare positions.

For the first occurrence:

```text id="8b7t9e"
sameer @ 0
khan   @ 1
```

and verify:

\[
pos(khan)=pos(sameer)+1
\]

Therefore the phrase occurs.

By contrast, if the document were:

```text id="zxg3uq"
sameer john khan
```

the positions would be:

```text id="gkwf7i"
sameer @ 0
john   @ 1
khan   @ 2
```

Now:

\[
pos(khan)\neq pos(sameer)+1
\]

so the exact phrase:

```text id="scgbo9"
"sameer khan"
```

does not occur.

This means phrase search can often be resolved using indexed positional information instead of rescanning the original document text.

---

#### Offsets

Offsets describe **character locations** in the original analyzed text rather than token ordinal positions.

For:

```text id="ddf46m"
sameer khan sameer
```

the character layout is approximately:

```text id="afyxks"
sameer khan sameer
012345678901234567
```

Conceptually:

```text id="xwwe4w"
sameer #1 → startOffset=0,  endOffset=6
khan      → startOffset=7,  endOffset=11
sameer #2 → startOffset=12, endOffset=18
```

Offsets therefore answer a different question from positions.

Positions answer:

> Which token number is this?

Offsets answer:

> Which character range in the original text produced this token?

A useful distinction is:

```text id="rzqvax"
positions → token-space coordinates
offsets   → character-space coordinates
```

---

#### Why keep both?

Positions are especially useful for:

```text id="hp6h6w"
phrase queries
proximity queries
ordered-token relationships
positional n-gram verification
```

Offsets are especially useful for:

```text id="xrcbx1"
highlighting matched text
mapping tokens back to source text
character-level span reconstruction
```

For example, after finding a match for `sameer`, offsets can tell the application which exact characters should be highlighted.

---

#### This becomes important again with n-grams

Later, if:

```text id="k1p5k4"
sameer
```

is converted into trigrams:

```text id="ndsqzo"
sam
ame
mee
eer
```

simply finding all four grams in the same document may produce false positives.

Position or offset information can help verify that the grams occur in the correct overlapping sequence:

```text id="45dg2h"
sam → starts at 0
ame → starts at 1
mee → starts at 2
eer → starts at 3
```

which reconstructs:

```text id="k14cg7"
sameer
```

So richer postings let an inverted index answer not only:

> Which documents contain this term?

but also:

> Where, how often, and in what relationship to other terms does it occur?  

## 11. Why Are Postings Sorted?

Suppose:

```text
sameer → [1, 3, 4, 6]
khan   → [1, 2, 5, 6]
```

Now query:

```text
sameer AND khan
```

The answer requires the intersection:

```text
[1, 3, 4, 6] ∩ [1, 2, 5, 6]
```

Because both lists are sorted, they can be intersected using advancing cursors:

```text
sameer     khan

1          1   → match
3          2   → advance khan
3          5   → advance sameer
4          5   → advance sameer
6          5   → advance khan
6          6   → match
```

Result:

```text
[1, 6]
```

For postings-list lengths `m` and `n`, the basic merge-style intersection is approximately:

```text
O(m + n)
```

Lucene contains additional optimizations, but sorted postings are the essential foundation.

---

## 12. Larger Toy Example

Assume:

```text
D1 = Sameer Khan
D2 = Sameer Ahmed
D3 = John Smith
D4 = Ahmed Khan
D5 = Sameer Smith
D6 = Samir Khan
D7 = John Sameer
D8 = Maria Khan
```

After analysis:

```text
D1 → sameer, khan
D2 → sameer, ahmed
D3 → john, smith
D4 → ahmed, khan
D5 → sameer, smith
D6 → samir, khan
D7 → john, sameer
D8 → maria, khan
```

The inverted index contains:

```text
ahmed  → [2, 4]
john   → [3, 7]
khan   → [1, 4, 6, 8]
maria  → [8]
sameer → [1, 2, 5, 7]
samir  → [6]
smith  → [3, 5]
```

Therefore:

```text
sameer → [1, 2, 5, 7]
```

and:

```text
df(sameer) = 4
```

---

## 13. Core Mental Model

For the exact query:

```text
sameer
```

think:

```text
            QUERY
           "sameer"
              │
              ▼
       Query Analyzer
              │
              ▼
           sameer
              │
              ▼
      TERM DICTIONARY
              │
       locate "sameer"
              │
              ▼
       POSTINGS LIST
         [1,2,5,7]
              │
              ▼
      matching documents
```

Do **not** primarily think:

```text
scan documents looking for strings
```

Think:

```text
find indexed term
    ↓
follow postings
```

---

## 14. The Most Important Distinction

Exact search contains two logically separate problems:

```text
1. Term discovery
2. Document discovery
```

For:

```text
sameer
```

term discovery is extremely constrained:

```text
find exactly one dictionary term
```

Once the term is found, document discovery is:

```text
read the term's postings list
```

This distinction becomes the foundation for understanding wildcard behavior.

For an exact query:

```text
sameer
```

the system asks:

```text
Where is the dictionary entry "sameer"?
```

For a wildcard query such as:

```text
sameer*
```

the question becomes:

```text
Which dictionary terms begin with "sameer"?
```

For:

```text
*sameer
```

the question becomes harder:

```text
Which dictionary terms end with "sameer"?
```

The postings mechanism remains recognizable.

What changes dramatically is **term discovery**.

---

## 15. Stage 1 Takeaways

### Inverted index

```text
Document → Terms
```

is transformed into:

```text
Term → Documents
```

### Exact search

```text
sameer
```

becomes:

```text
query analysis
    ↓
term dictionary lookup
    ↓
postings lookup
    ↓
matching DocIDs
```

### First-order cost model

```text
T_query
  ≈
T_term_lookup
  +
T_postings_processing
```

where postings-processing cost is strongly influenced by:

```text
df(term)
```

### Core performance insight

An exact-term query does not require scanning all documents.

The inverted index allows the engine to jump from:

```text
term
```

directly to:

```text
documents containing that term
```

### Foundation for the next stages

The key distinction to preserve is:

```text
term discovery ≠ document discovery
```

For exact search, term discovery is easy.

Wildcard search becomes interesting because it makes term discovery progressively harder.

---

## Stage 1 Checkpoint

I should now be able to explain:

1. Why an inverted index exists.
2. Why it is called "inverted."
3. The difference between the term dictionary and postings lists.
4. How `sameer` reaches `[DocID...]` without scanning every document.
5. What document frequency `df(term)` means.
6. Why postings lists are sorted.
7. Why exact-query cost depends significantly on postings size rather than only total corpus size.
8. Why `term discovery` and `document discovery` are separate algorithmic problems.

Next:

```text
Stage 2 — Prefix Search: sameer*
```

The goal will be to understand why the sorted term space makes prefix lookup fundamentally easier than a leading wildcard such as:

```text
*sameer
```

## Sorted Postings, Segments, and Immutable Index Structures

A simple educational model for an inverted index is:

```java
Map<String, List<Posting>> invertedIndex;
```

where each term maps to a postings list containing entries such as:

```text
Posting
- docId
- termFrequency
- positions
- offsets
```

For example:

```text
sameer → [1, 3, 7, 12]
```

The postings list should remain sorted by `docId`.

This sorted ordering is important because later query operations such as Boolean intersection and union can exploit monotonic DocID order.

---

### The Write-Side Problem with Sorted Arrays

If DocIDs arrive monotonically, adding a new posting is cheap:

```text
[1, 3, 7, 12] + 15

→ [1, 3, 7, 12, 15]
```

An array-backed structure such as `ArrayList` can append efficiently.

However, arbitrary insertion is more expensive.

Suppose we need to insert DocID `5`:

```text
Before:

[1, 3, 7, 12]

After:

[1, 3, 5, 7, 12]
```

Binary search can find the insertion point in:

\[
O(\log n)
\]

but the remaining array elements must be shifted, which costs:

\[
O(n)
\]

Therefore:

\[
T_{insert}
\approx
O(\log n) + O(n)
=
O(n)
\]

For very large postings lists, continuously mutating sorted arrays would be undesirable.

---

## Lucene's Segment-Based Approach

Lucene avoids maintaining one giant mutable inverted index.

Instead, the index is divided into **segments**.

Conceptually:

```text
Index
├── Segment A
│   ├── Term Dictionary
│   └── Postings
│
├── Segment B
│   ├── Term Dictionary
│   └── Postings
│
└── Segment C
    ├── Term Dictionary
    └── Postings
```

Each segment is approximately a small independent inverted index.

New documents are accumulated in memory and eventually flushed as a new segment.

For example:

```text
Segment A

sameer → [0, 2]
khan   → [1, 2]
```

Later documents may create:

```text
Segment B

sameer → [0, 1]
lucene → [1, 3]
```

Rather than modifying Segment A in place, Lucene writes another immutable segment.

This changes the write problem from:

```text
arbitrary insertion into existing sorted arrays
```

into:

```text
build new sorted structures
+
merge them later
```

This is a much more efficient systems pattern.

---

## Segments Are Immutable

Once written, a segment is generally treated as immutable.

Conceptually:

```text
new documents
      ↓
mutable in-memory indexing state
      ↓
flush
      ↓
immutable segment
```

Over time:

```text
Segment A
Segment B
Segment C
...
```

This design is conceptually similar to the broader systems pattern used by structures such as LSM trees and SSTables:

```text
write new sorted structures
→ keep them immutable
→ periodically merge them
```

The implementation details differ, but the underlying idea is similar.

---

## DocIDs Are Segment-Local

An important refinement to the simple postings model is that Lucene DocIDs are local to a segment.

For example:

```text
Segment A:
local DocIDs = 0, 1, 2

Segment B:
local DocIDs = 0, 1, 2, 3
```

The same local DocID can therefore exist in multiple segments.

The effective identity is approximately:

```text
(segment, localDocID)
```

At query time, Lucene can conceptually assign each segment a `docBase`.

Example:

```text
Segment A
docBase = 0
local DocIDs = 0,1,2

Segment B
docBase = 3
local DocIDs = 0,1,2,3
```

A global query-time DocID can be viewed as:

globalDocID = docBase + localDocID  

For example:

```text
Segment A, local 2 → global 2
Segment B, local 2 → global 5
```

---

## Exact Term Lookup Across Segments

Suppose the index contains:

```text
Segment A
sameer → [0, 2]

Segment B
sameer → [1, 3]

Segment C
sameer → [0]
```

A query for:

```text
sameer
```

is conceptually evaluated independently against each segment:

```text
Segment A
    lookup "sameer"
    traverse postings [0,2]

Segment B
    lookup "sameer"
    traverse postings [1,3]

Segment C
    lookup "sameer"
    traverse postings [0]
```

The results are then combined logically.

If:

```text
A.docBase = 0
B.docBase = 3
C.docBase = 7
```

then:

```text
A [0,2] → [0,2]
B [1,3] → [4,6]
C [0]   → [7]
```

Conceptual combined result:

```text
[0, 2, 4, 6, 7]
```

---

## Query Cost with Multiple Segments

For a single simplified inverted index, the exact-term lookup cost was:
`T_query = T_termlookup + T_postings`  

With multiple segments:
T_query ≈ Σ(s=1 to S) (T_termLookup,s + T_postings,s)

where:

```text
S = number of segments
```

Ignoring optimizations such as caching, parallelism, skipping, and scoring, this exposes an important performance property:

> More segments introduce more per-query fixed overhead.

Even if a term does not exist in a particular segment, the segment must still be checked.

---

## Why Segment Merging Is Necessary

If indexing continuously creates small segments:

```text
S0
S1
S2
S3
...
S99
```

then every query may need to consult many term dictionaries.

Lucene therefore periodically merges segments.

Suppose:

```text
Segment A

sameer → [0, 2]
```

and:

```text
Segment B

sameer → [1, 3]
```

Assume Segment A contains three documents.

During a merge, Segment B's DocIDs can be remapped after Segment A:

```text
Segment A:
[0, 2]

Segment B:
local [1, 3]

offset by 3:

[4, 6]
```

The merged postings become:

```text
sameer → [0, 2, 4, 6]
```

---

## Sorted Postings Make Merging Efficient

Because both input postings lists are already sorted, merging can use the classic two-pointer merge algorithm.

Example:

```text
A = [0, 2, 5, 9]
B = [1, 3, 7, 10]
```

Walk both lists in increasing order:

```text
A pointer →
B pointer →
```

and produce:

```text
[0, 1, 2, 3, 5, 7, 9, 10]
```

The merge cost is:

\[
O(|A| + |B|)
\]

This is significantly better than repeatedly performing arbitrary insertions into a large existing array.

---

## Sortedness Pays Twice

Sorted postings are valuable on both the read path and the maintenance path.

### Query side

```text
sorted postings
      ↓
efficient intersection
efficient union
efficient skipping
```

For example:

```text
A = [1, 3, 7, 10]
B = [2, 3, 7, 11]
```

The intersection can be found with a linear merge-style traversal:

```text
A AND B → [3, 7]
```

### Merge side

```text
sorted segment postings
      ↓
efficient segment merging
```

Therefore sorted DocIDs are not merely a query optimization.

They are a fundamental index-structure invariant that also enables efficient maintenance.

---

## A Better Educational Data Model

The original simplified structure:

```java
Map<String, List<Posting>> invertedIndex;
```

can now be refined to:

```java
class Segment {
    Map<String, List<Posting>> invertedIndex;
}
```

and the complete index can be viewed conceptually as:

```java
class Index {
    List<Segment> segments;
}
```

So:

```text
Index
  ↓
List<Segment>

Segment
  ↓
Term Dictionary
  ↓
Postings Lists
```

Within each segment:

```text
terms are searchable in the term dictionary

and

postings DocIDs are monotonically increasing
```

There is no requirement for one continuously maintained global postings list.

---

## Updates and Deletes

Segment immutability also changes how updates work.

Suppose an existing document contains:

```text
sameer khan
```

and is updated to:

```text
sameer ahmed
```

Lucene cannot simply mutate the old immutable segment and remove the document from:

```text
khan → [...]
```

Conceptually, the update behaves more like:

```text
delete old document
+
add new document
```

The old document can be marked logically deleted:

```text
Segment A

Doc 0  live
Doc 1  live
Doc 2  deleted
Doc 3  live
```

A future segment merge can omit deleted documents when constructing the new segment.

Thus merging also acts as a form of physical cleanup.

---

## Performance Engineering Implications

Segment architecture introduces an important system-level tradeoff.

### Frequent Small Flushes

```text
Advantages:
- smaller individual flushes
- less in-memory indexing state required

Disadvantages:
- more segments
- more term-dictionary lookups per query
- more future merge work
```

### Larger, Less Frequent Flushes

```text
Advantages:
- fewer segments
- lower per-query segment overhead

Disadvantages:
- larger indexing buffers
- larger individual flush operations
```

### Merge Cost

Merges consume resources:

```text
read old segment data
+
CPU processing
+
write new segment data
+
storage bandwidth
+
cache pressure
```

Therefore the index has a fundamental systems tradeoff:  
`indexing throughput <-> merge cost <-> segment count <-> query latency`  
---

## Core Mental Model

Lucene avoids continuously mutating one giant global postings structure.

Instead:

```text
documents arrive
      ↓
build postings in memory
      ↓
flush a new immutable sorted segment
      ↓
query across multiple segments
      ↓
periodically merge segments
```

The key architectural insight is:  
*Immutable sorted segments transform costly random mutation into sequential construction and efficient merging. This preserves sorted postings while supporting continuous indexing and efficient querying.*   

