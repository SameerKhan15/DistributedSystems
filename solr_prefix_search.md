## Stage 2 — Prefix Search: `sameer*`

A prefix query such as:

```text
sameer*
```

means:

```text
Find all indexed terms whose prefix is "sameer".
```

Example term dictionary:

```text
sam
same
sameer
sameera
sameerahmed
sameerkhan
sameers
samir
samuel
```

Matching terms are:

```text
sameer
sameera
sameerahmed
sameerkhan
sameers
```

### 1. Why Lexicographic Ordering Makes Prefix Search Efficient

Lucene stores terms in lexicographic order.

All terms sharing the same leading prefix therefore occupy a contiguous region of term space:

```text
same
-------------------
sameer
sameera
sameerahmed
sameerkhan
sameers
-------------------
samir
samuel
```

Conceptually, prefix search becomes:

```text
seek to first term >= "sameer"

then enumerate forward

while term starts with "sameer"
```

Example:

```text
seek("sameer")
     ↓
sameer        ✓
sameera       ✓
sameerahmed   ✓
sameerkhan    ✓
sameers       ✓
samir         ✗ STOP
```

The key property is:

```text
leading prefix
    +
lexicographic ordering
    ↓
contiguous dictionary region
```

---

### 2. Abstract Meaning of `seek`

At the abstract level:

```text
seekCeil("sameer")
```

means:

```text
Position the term enumerator at the first dictionary term
whose value is >= "sameer".
```

This is still **term discovery**, not document discovery.

```text
term discovery
    ≠
document discovery
```

The goal of the seek is to efficiently locate the relevant portion of the term dictionary.

---

### 3. Lucene BlockTree and the FST Term Index

Lucene uses a BlockTree term dictionary.

Conceptually:

```text
FST term index
      ↓
BlockTree block / sub-block / floor block
      ↓
TermsEnum
      ↓
matching terms
```

The FST helps Lucene avoid scanning the dictionary from the beginning.

For a prefix such as:

```text
sameer
```

the bytes are consumed deterministically:

```text
s → a → m → e → e → r
```

During traversal, outputs attached to FST arcs are accumulated.

Conceptually:

```text
currentState
accumulatedOutput
```

The result is navigation metadata that helps Lucene locate the relevant BlockTree region.

Important:

```text
FST output ≠ postings
FST output ≠ DocIDs
```

A better model is:

```text
term prefix
    ↓
FST traversal
    ↓
BlockTree navigation metadata
    ↓
relevant .tim block / floor block
```

---

### 4. FST Relationship to a Trie

An FST builds on the same prefix-sharing idea as a trie.

Conceptually:

```text
Trie
  ↓
share common prefixes
  ↓
finite-state representation
  ↓
minimize equivalent states
  ↓
add outputs
  ↓
FST
```

A trie might represent:

```text
same
sameer
sameera
sameerkhan
```

as shared prefix paths:

```text
s
└── a
    └── m
        └── e
            └── e
                └── r
                    ├── a...
                    └── k...
```

A minimized finite-state structure can additionally merge states that have identical future behavior.

An FST then associates outputs with transitions.

In Lucene, those outputs help encode navigation information for the BlockTree term dictionary.

Therefore, a useful mental model is:

```text
Trie idea:
share prefix bytes

FST:
compressed/minimized prefix structure
+
outputs
```

---

### 5. Floor Blocks

A BlockTree region associated with one prefix may still contain too many terms.

Lucene can divide such a region into floor blocks.

Conceptually:

```text
same...
   │
   ├── floor block A
   ├── floor block B
   └── floor block C
```

Additional bytes from the seek term can help select the appropriate floor block.

This avoids scanning a very large logical prefix block unnecessarily.

---

### 6. TermsEnum Enumeration

Once Lucene reaches the relevant BlockTree region, `TermsEnum` performs term enumeration.

Conceptually:

```text
TermsEnum cursor
       ↓
sameer
sameera
sameerahmed
sameerkhan
sameers
samir
```

For `sameer*`:

```text
sameer        ✓
sameera       ✓
sameerahmed   ✓
sameerkhan    ✓
sameers       ✓
samir         ✗
```

`TermsEnum` is enumerating dictionary terms, not documents.

---

### 7. Query Automaton vs Term-Index FST

There are two separate automata-like structures involved.

#### Term-index FST

Purpose:

```text
Where in the term dictionary should I look?
```

#### Prefix query automaton

Purpose:

```text
Which terms are accepted?
```

For:

```text
sameer*
```

the accepted language is conceptually:

```text
sameer Σ*
```

where `Σ*` means zero or more following symbols.

Examples:

```text
sameer        ✓
sameera       ✓
sameerkhan    ✓

same          ✗
samir         ✗
```

Conceptually, Lucene can combine:

```text
dictionary term structure
        ∩
query automaton
```

to avoid visiting dictionary regions that cannot contain matching terms.

---

### 8. From Matching Terms to Documents

Suppose the prefix matches:

```text
sameer
sameera
sameerkhan
```

with postings:

```text
sameer
→ [1, 8, 20]

sameera
→ [3, 9]

sameerkhan
→ [2, 8, 17]
```

Semantically, the query matches the union:

```text
{1, 8, 20}
∪
{3, 9}
∪
{2, 8, 17}
```

Result:

```text
{1, 2, 3, 8, 9, 17, 20}
```

Lucene does not necessarily materialize this as thousands of ordinary Boolean clauses. Multi-term queries use rewrite/execution strategies optimized for this kind of expansion.

However, the logical interpretation remains:

```text
OR the postings of all accepted terms
```

---

### 9. Prefix Search Cost Model

A useful first-order model is:

```text
T_prefix
≈
T_seek
+
T_termEnumeration
+
T_postings
```

Let:

```text
M = number of matching dictionary terms
```

Then:

```text
T_prefix
≈
T_seek
+
O(M)
+
Σ T(postings_i)
```

If:

```text
df_i = document frequency of matching term i
```

then a rough conceptual model is:

```text
T_prefix
≈
T_seek
+
O(M)
+
O(Σ df_i)
```

This is a teaching model rather than an exact Lucene latency equation.

Actual execution is also influenced by:

```text
segment count
query rewrite strategy
scoring mode
postings encoding
skip data
cache behavior
top-K collection
duplicate DocIDs across terms
```

---

### 10. Term Expansion vs Postings Expansion

These are independent dimensions.

Example A:

```text
M = 3 matching terms
```

but:

```text
df(sameer) = 10,000,000
```

Term expansion is small, but postings work may be huge.

Example B:

```text
M = 50,000 matching terms
```

but each term appears in only one or two documents.

Term expansion is large, while postings may still be relatively sparse.

Therefore:

```text
term expansion
    ≠
postings expansion
```

Both matter to prefix-query performance.

---

### 11. Why Prefix Selectivity Matters

Compare:

```text
s*
```

with:

```text
sameer*
```

The seek cost may be relatively similar in scale.

The major difference is typically the amount of expansion after localization.

For example:

```text
M("s*")      = 1,000,000
M("sameer*") = 20
```

Therefore:

```text
T_enum("s*")
≫
T_enum("sameer*")
```

and often:

```text
T_postings("s*")
≫
T_postings("sameer*")
```

This suggests a useful decomposition:

```text
Prefix query cost
=
localization cost
+
expansion cost
```

where:

```text
localization
≈ FST + BlockTree seek

expansion
≈ term enumeration + postings processing
```

For broad prefixes, expansion usually dominates localization.

---

### 12. Why `sameer*` Is Easier Than `*sameer`

For:

```text
sameer*
```

all matching terms share leading bytes:

```text
sameer...
```

Lexicographic ordering places them together:

```text
leading prefix
    ↓
contiguous term region
```

For:

```text
*sameer
```

matching terms may appear anywhere:

```text
asameer
johnsameer
ksameer
mysameer
sameer
xsameer
```

They do not form one contiguous lexical interval because dictionary ordering is based on the beginning of each term.

Therefore:

```text
sameer*
```

aligns naturally with the organization of the term dictionary.

```text
*sameer
```

does not.

The deeper reason leading wildcards are expensive is not simply that "wildcards are slow."

It is:

```text
A leading wildcard removes the prefix information
that the sorted term dictionary is organized around.
```

---

### 13. End-to-End Mental Model

```text
Query: sameer*
        ↓
prefix/multi-term query
        ↓
query automaton accepts sameer...
        ↓
lexicographically ordered term dictionary
        ↓
FST traversal using prefix bytes
        ↓
accumulate BlockTree navigation metadata
        ↓
enter relevant block / sub-block / floor block
        ↓
TermsEnum enumerates accepted terms
        ↓
sameer
sameera
sameerahmed
sameerkhan
sameers
        ↓
process postings for accepted terms
        ↓
combine matching documents
        ↓
matching DocIDs
```

### Core Stage 2 Insight

A prefix query is efficient because:

```text
prefix
+
lexicographic ordering
=
contiguous term-space region
```

Lucene's BlockTree and FST term index help efficiently locate that region.

After localization, query cost is largely determined by:

```text
number of matching terms
+
size of their postings
```

A compact performance model is:

```text
T_prefix
≈
T_seek
+
O(M)
+
Σ T(postings_i)
```

where:

```text
M = number of terms matching the prefix
```

The most important distinction to preserve is:

```text
FST navigation
    ↓
find term region

TermsEnum
    ↓
discover matching terms

Postings
    ↓
discover matching documents
```