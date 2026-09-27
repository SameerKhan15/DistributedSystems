## Stage 6 — Unified Cost Model and Query-Shape Comparison

A useful unified model for index-based query cost is:

```text
T_query
≈
T_termDiscovery
+
T_postings
+
T_setOperations
+
T_verification
```

Where:

```text
V = number of unique indexed terms
M = number of dictionary terms matching the query
df(t) = number of documents containing term t
P = Σ df(t) across matching terms

For n-grams:
G = number of query grams
C = candidate documents surviving gram intersection
```

The key idea is that **term-discovery cost and postings cost are separate dimensions**. Two queries can have identical `M` and `P` yet have very different costs because finding those `M` terms may require very different amounts of dictionary exploration.

### Exact Search — `sameer`

```text
exact lookup
    ↓
postings(sameer)
```

```text
T_exact
≈
T_seek
+
T_postings
```

Term discovery is highly targeted (`M = 1`), but postings can still dominate if the term has high document frequency.

---

### Prefix Search — `sameer*`

Terms sharing a prefix occupy a contiguous lexicographic region:

```text
seek("sameer")
      ↓
enumerate prefix range
      ↓
postings
```

```text
T_prefix
≈
T_seek
+
O(M)
+
O(P)
```

Prefix selectivity matters:

```text
M(s*) >> M(sameer*)
```

Both are prefix queries, but `s*` may expand into vastly more terms.

---

### Leading Wildcard — `*sameer`

The fixed information is a suffix and therefore does not align well with forward lexicographic ordering.

```text
broad dictionary exploration
        ↓
matching terms
        ↓
postings
```

```text
T_leadingWildcard
≈
T_dictionaryExploration
+
O(M)
+
O(P)
```

Therefore:

```text
sameer* → M = 10, P = 1000
*sameer → M = 10, P = 1000
```

does **not** imply equal cost. The leading wildcard may require substantially more work to discover those same 10 terms.

---

### Arbitrary Substring — `*sameer*`

There is no fixed prefix and no single narrow lexicographic region.

```text
broad dictionary exploration
        ↓
matching terms
        ↓
postings
```

```text
T_substring
≈
T_broadDictionaryExploration
+
O(M)
+
O(P)
```

A query can return very few documents and still be expensive:

```text
result count ≠ query cost
```

The expensive work may happen while discovering terms rather than processing the final result set.

---

### N-Gram Substring Search

With trigrams:

```text
sameer
  ↓
sam
ame
mee
eer
```

Substring discovery becomes several exact lookups:

```text
exact lookup(sam)
exact lookup(ame)
exact lookup(mee)
exact lookup(eer)
        ↓
postings intersection
        ↓
C candidate documents
        ↓
optional positional/offset verification
```

```text
T_ngram
≈
Σ T_exactLookup(gram_i)
+
T_postingsIntersection
+
T_verification
```

N-grams therefore replace broad wildcard dictionary exploration with multiple targeted exact lookups.

### Union vs. Intersection

Wildcard expansion is conceptually OR-like:

```text
term1 OR term2 OR term3
```

and therefore involves postings **union**.

N-gram decomposition is conceptually AND-like:

```text
sam AND ame AND mee AND eer
```

and therefore involves postings **intersection**.

Intersection ordering matters. Given:

```text
sam → 2,000,000 docs
ame →   700,000 docs
mee →    40,000 docs
eer →   900,000 docs
```

starting with the selective `mee` postings can shrink the candidate set early, analogous to database join ordering.

---

### Term Selectivity vs. Document Selectivity

These are different dimensions:

```text
term selectivity
≈ M / V

document selectivity
≈ matchingDocs / totalDocs
```

A query such as:

```text
*rare-substring*
```

may have excellent document selectivity while still requiring expensive dictionary exploration.

Similarly, dictionary and postings costs are independent:

```text
cheap discovery     + small postings
cheap discovery     + huge postings
expensive discovery + small postings
expensive discovery + huge postings
```

---

### Read/Write Amplification

For token length `L` and gram size `n`:

```text
G = L - n + 1
```

For example:

```text
L = 20
n = 3

G = 18 trigrams
```

N-gram indexing therefore increases index size, postings volume, indexing work, and merge work.

Conceptually:

```text
W_ngram > W_normal
```

but for substring-heavy workloads:

```text
R_ngram < R_wildcard
```

This is a classic:

```text
write amplification
        ↕
read amplification
```

tradeoff, similar to secondary indexes, materialized views, and denormalization.

---

### Unified Query-Shape Model

| Query | Term Discovery | Dictionary Locality | Postings Work |
|---|---|---|---|
| `sameer` | exact lookup | excellent | one list |
| `sameer*` | prefix-range enumeration | excellent | multiple lists |
| `*sameer` | broad exploration | poor | multiple lists |
| `*sameer*` | broad substring exploration | very poor | multiple lists |
| n-gram substring | multiple exact lookups | excellent | intersection + verification |

The general hierarchy is:

```text
exact
  ↓
prefix
  ↓
suffix
  ↓
arbitrary substring
```

As the query becomes less aligned with forward lexicographic ordering, term discovery generally becomes harder.

N-gram indexing changes the representation so arbitrary substring information becomes accessible through exact lookup keys.

### Core Principle

> **An index is not generically "fast search." It is fast for access patterns that align with its representation.**

Therefore:

```text
query performance
=
relationship between
query shape
and
indexed representation
```

When diagnosing an unfamiliar query, reason separately about:

```text
1. How expensive is term discovery?
2. How many terms match?                 M
3. How much postings data is behind them? P
4. Are postings unioned or intersected?
5. How many candidates survive?          C
6. Is verification required?
```

This identifies whether latency is primarily **dictionary-bound, postings-bound, set-operation-bound, or verification-bound**, rather than incorrectly using final result count as a proxy for query cost.  

