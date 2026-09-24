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

## 10. Why Are Postings Sorted?

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

## 11. Larger Toy Example

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

## 12. Core Mental Model

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

## 13. The Most Important Distinction

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

## 14. Stage 1 Takeaways

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
