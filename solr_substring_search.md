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
