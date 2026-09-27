## Stage 4 — Substring Search: `*sameer*`

### Semantics

```text
*sameer*
```

matches any indexed term containing `sameer` at any position:

```text
sameer
sameerkhan
johnsameer
johnsameerkhan
customer_sameer_archive
x_sameer_y
```

Formally:

```text
Σ* sameer Σ*
```

where `Σ*` represents zero or more arbitrary characters.

---

### Why Substring Search Is Expensive

A normal Lucene term dictionary is lexicographically ordered.

For a prefix query:

```text
sameer*
```

the fixed prefix identifies a narrow, contiguous region of term space:

```text
sameer
sameera
sameerkhan
sameers
```

But for:

```text
*sameer*
```

matching terms can begin with anything:

```text
asameerb
customer_sameer_archive
johnsameer
presameerpost
sameer
xsameery
```

These terms are scattered throughout the dictionary.

The query provides no fixed beginning from which Lucene can efficiently seek.

Conceptually, the wildcard automaton is:

```text
ANYTHING*
    ↓
s → a → m → e → e → r
    ↓
ANYTHING*
```

Because almost any initial dictionary branch could eventually contain `sameer`, early pruning is much weaker than for a prefix query.

---

### Three Sources of Query Work

Substring wildcard cost can be decomposed conceptually into:

```text
T_substring
≈
T_dictionaryExploration
+
T_termExpansion
+
T_postings
```

#### Dictionary exploration

Find dictionary terms accepted by the wildcard automaton.

#### Term expansion

Identify the concrete indexed terms matching `*sameer*`.

For example:

```text
*sameer*
    ↓
asameerb
johnsameer
presameerpost
sameer
xsameery
```

#### Postings expansion

Process the postings associated with those matching terms to determine matching documents.

This explains why:

```text
result count ≠ query cost
```

A query might return only a few documents while requiring substantial dictionary exploration to discover the matching terms.

---

### Why Reversing Terms Does Not Solve It

Reversing terms is useful for suffix search:

```text
*sameer
```

because reversing transforms it into:

```text
reemas*
```

which becomes a prefix search.

But:

```text
*sameer*
```

becomes:

```text
*reemas*
```

after reversal.

It remains a substring search.

Therefore:

```text
sameer*      → forward prefix representation works

*sameer      → reversed prefix representation works

*sameer*     → neither orientation provides a fixed prefix
```

---

## N-Gram Indexing

Substring search motivates changing the indexed representation.

Suppose we use trigrams (`n = 3`).

For:

```text
johnsameerkhan
```

generate:

```text
Position    Gram

0           joh
1           ohn
2           hns
3           nsa
4           sam
5           ame
6           mee
7           eer
8           erk
9           rkh
10          kha
11          han
```

These grams become searchable indexed terms.

The substring:

```text
sameer
```

can itself be decomposed into:

```text
sam
ame
mee
eer
```

Instead of broadly exploring the dictionary for:

```text
*sameer*
```

we can perform exact indexed lookups:

```text
lookup("sam")
lookup("ame")
lookup("mee")
lookup("eer")
```

and intersect their postings:

```text
Docs(sam)
∩
Docs(ame)
∩
Docs(mee)
∩
Docs(eer)
```

This produces candidate documents.

---

### Why Gram Intersection Alone Is Not Enough

Finding all four grams in one document does not necessarily prove that `sameer` occurs contiguously.

For example:

```text
sam @ position 2
ame @ position 17
mee @ position 31
eer @ position 50
```

satisfies:

```text
sam AND ame AND mee AND eer
```

but does not establish the contiguous substring:

```text
sameer
```

The grams must occur in the correct adjacent positions.

For a real `sameer` occurrence:

```text
sam @ p
ame @ p+1
mee @ p+2
eer @ p+3
```

For example:

```text
johnsameerkhan

sam @ 4
ame @ 5
mee @ 6
eer @ 7
```

The overlapping grams reconstruct:

```text
sam
 ame
  mee
   eer
------
sameer
```

Therefore, depending on the Lucene/Solr indexing and query strategy, positional information can be used to enforce this relationship, or gram matching can generate candidates followed by verification against the original value.

Conceptually:

```text
*sameer*
    ↓
sam + ame + mee + eer
    ↓
exact gram lookups
    ↓
postings intersection
    ↓
candidate documents
    ↓
positional / original-value verification
    ↓
true substring matches
```

---

### N-Gram Cost Model

Conceptually:

```text
T_ngram
≈
T_gramLookups
+
T_postingsIntersection
+
T_verification
```

N-grams therefore shift work from broad query-time term discovery toward index-time preprocessing.

The trade-off is:

```text
More index storage
More terms/postings
More indexing CPU
        ↓
Faster and more predictable substring discovery
```

The choice of `n` matters. Smaller grams tend to occur more frequently and therefore produce larger postings lists and more candidate matches.

---

### Core Insight

A normal term dictionary organizes whole terms primarily by their prefixes.

But:

```text
*sameer*
```

asks about characters occurring at an arbitrary internal position.

N-gram indexing changes the representation so that internal fragments themselves become indexed terms:

```text
Normal representation:

whole terms
    ↓
*sameer*
    ↓
broad dictionary exploration
```

versus:

```text
N-gram representation:

sam → postings
ame → postings
mee → postings
eer → postings
    ↓
postings intersection
    ↓
contiguity verification
```

The fundamental systems principle is:

> When a query cannot efficiently exploit the existing index organization, change the indexed representation so that the information required by the query becomes directly searchable.

## Stage 5 — The N-Gram Alternative

Arbitrary substring queries such as:

```text
*sameer*
```

are expensive because the fixed string `sameer` can occur anywhere inside an indexed term. Matching terms may therefore be scattered throughout the lexicographically ordered term dictionary.

Conceptually:

```text
*sameer*
   ↓
broad dictionary exploration
   ↓
matching-term discovery
   ↓
postings
```

N-grams take a different approach:

> Instead of making substring discovery cheaper at query time, change the indexed representation so substrings are searchable directly.

### N-Gram Indexing

An n-gram is a contiguous sequence of `n` characters.

For:

```text
sameer
```

using trigrams (`n = 3`):

```text
sam
ame
mee
eer
```

For a token of length `L` and fixed gram size `n`:

```text
G = L - n + 1
```

For `sameer`:

```text
L = 6
n = 3

G = 6 - 3 + 1 = 4
```

N-grams do **not** replace the inverted index. They change what is inserted into it.

Instead of only:

```text
sameer → postings
```

the index can contain:

```text
sam → postings
ame → postings
mee → postings
eer → postings
```

The fundamental structure remains:

```text
term → postings
```

The terms now happen to be character substrings.

---

### Toy Example

Consider:

```text
D1 = sameer
D2 = johnsameerkhan
D3 = samuel
D4 = ameer
```

Trigram indexing produces, among others:

```text
sam → [D1, D2, D3]
ame → [D1, D2, D4]
mee → [D1, D2, D4]
eer → [D1, D2, D4]
```

The substring query:

```text
*sameer*
```

can be decomposed into:

```text
sam
ame
mee
eer
```

Candidate retrieval becomes:

```text
postings(sam)
   ∩
postings(ame)
   ∩
postings(mee)
   ∩
postings(eer)
```

For the example:

```text
[D1,D2,D3]
    ∩
[D1,D2,D4]
    ∩
[D1,D2,D4]
    ∩
[D1,D2,D4]

    ↓

[D1,D2]
```

The key transformation is:

```text
arbitrary substring discovery
            ↓
exact gram lookups + postings intersection
```

This aligns substring search with the natural strengths of an inverted index.

---

### Candidate Retrieval vs. Contiguous Match

Finding every query gram in a document does not necessarily prove that the complete substring occurs contiguously.

For example:

```text
samxxxamexxxmeexxxeer
```

contains all four grams:

```text
sam
ame
mee
eer
```

but does not contain:

```text
sameer
```

Therefore:

```text
gram presence
     ↓
candidate document
```

does not necessarily imply:

```text
contiguous substring match
```

A system requiring exact substring semantics needs enough information to verify the required overlap.

Conceptually, `sameer` has:

```text
sam → start 0
ame → start 1
mee → start 2
eer → start 3
```

A valid overlapping sequence obeys:

```text
start(ame) = start(sam) + 1
start(mee) = start(sam) + 2
start(eer) = start(sam) + 3
```

Positions and offsets can provide information useful for such relationships:

```text
positions → token-space coordinates
offsets   → character-space coordinates
```

They should not be conflated. In particular, the character start offset of an n-gram is not automatically its Lucene token position; analyzer/tokenizer behavior determines how these attributes are emitted.

Depending on the indexing/query design, exact verification may use positional/offset information or verification against the original value.

---

### Gram Size Tradeoff

For `sameer`:

```text
n = 2:

sa
am
me
ee
er
```

```text
n = 3:

sam
ame
mee
eer
```

```text
n = 4:

same
amee
meer
```

Smaller `n`:

```text
+ supports shorter substring queries
+ provides finer-grained matching

- produces more grams
- grams tend to be less selective
- postings lists can be larger
- more candidate documents / false positives
```

Larger `n`:

```text
+ produces fewer grams
+ grams tend to be more selective
+ postings lists can be smaller
+ fewer candidate documents

- cannot directly represent queries shorter than n
```

Therefore `n` is a workload-dependent tuning parameter.

---

### N-Gram vs. Edge N-Gram

Normal n-grams slide across the complete token:

```text
sameer
   ↓
sam
ame
mee
eer
```

They are useful for arbitrary substring access.

Edge n-grams are anchored to an edge:

```text
sameer
   ↓
s
sa
sam
same
samee
sameer
```

They are primarily useful for prefix/autocomplete access.

Mental model:

```text
Edge n-gram ≈ prefix acceleration

N-gram      ≈ arbitrary substring acceleration
```

---

### Index Amplification

N-grams shift work from query time to index time.

For a token of length `L`:

```text
G = L - n + 1
```

For:

```text
L = 20
n = 3
```

one token produces:

```text
18 trigrams
```

At scale this can mean:

```text
more gram occurrences
      ↓
larger postings structures
      ↓
larger index
      ↓
more indexing work
      ↓
more segment-merge work
      ↓
more cache / memory / I/O pressure
```

N-grams therefore trade write/storage amplification for cheaper substring discovery.

---

### Query Cost

Without n-grams:

```text
T_substring
≈
T_broadDictionaryExploration
+
T_termExpansion
+
T_postings
```

With n-grams:

```text
T_ngram
≈
Σ T_exactLookup(gram_i)
+
T_postingsIntersection
+
T_verification
```

The important transformation is replacing broad dictionary exploration with targeted exact term lookups.

---

### Postings Intersection Ordering

Not all grams are equally selective.

Suppose:

```text
sam → 2,000,000 docs
ame →   500,000 docs
mee →    50,000 docs
eer →   700,000 docs
```

The most selective gram is:

```text
mee → 50,000 docs
```

Starting from a selective postings list can reduce the candidate population early:

```text
mee
 ↓
∩ ame
 ↓
∩ eer
 ↓
∩ sam
```

This follows the same general principle as database join ordering:

> Reduce intermediate candidate sets as early as possible.

Document frequency provides a useful measure:

```text
df(gram) = number of documents containing the gram
```

Lower `df` generally means greater selectivity.

---

### Architectural Tradeoff

Without n-grams:

```text
smaller/simpler index
        +
potentially expensive substring discovery
```

With n-grams:

```text
more indexing work
+
larger index
+
more postings
+
more merge/cache/I/O pressure
        ↓
targeted exact lookups
+
postings intersection
+
optional verification
```

The deeper systems principle is:

```text
expensive repeated query-time computation
                ↓
precompute additional searchable representation
                ↓
pay write + storage cost
                ↓
make repeated reads cheaper
```

This is analogous to:

- secondary indexes
- materialized views
- denormalization
- pre  


