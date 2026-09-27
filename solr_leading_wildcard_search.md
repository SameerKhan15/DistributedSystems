## Stage 3 — Leading Wildcard Search: `*sameer`

A leading wildcard query:

```text
*sameer
```

means:

> Find every indexed term that **ends with** `sameer`.

Examples:

```text
sameer            ✓
asameer           ✓
customersameer    ✓
johnsameer        ✓
systemsameer      ✓
xsameer           ✓

sameera           ✗
sameerkhan        ✗
sameerengineer    ✗
samir             ✗
```

### 1. Why Leading Wildcards Are Different

Lucene's term dictionary is organized according to the **beginning of terms**.

For a prefix query:

```text
sameer*
```

the query immediately provides useful navigation information:

```text
s → a → m → e → e → r
```

Terms sharing this prefix occupy a narrow, contiguous region of lexicographic term space.

Conceptually:

```text
seek("sameer")
      ↓
enumerate matching terms
      ↓
stop when prefix changes
```

For:

```text
*sameer
```

the beginning of the term is unknown.

Matching terms may occur throughout the dictionary:

```text
asameer
bob
customersameer
john
johnsameer
khan
sameer
smith
systemsameer
xsameer
```

There is no single narrow lexicographic region containing all suffix matches.

The query still contains useful information (`sameer`), but that information is oriented from the **wrong end** of the indexed structure.

---

### 2. No Useful Root-Level Prefix Pruning

Consider a conceptual prefix-oriented term structure:

```text
             ROOT
       /   /   |   \   \
      a   b    c   ...  x
```

For:

```text
sameer*
```

only the `s` branch can match.

Most root branches can immediately be eliminated.

For:

```text
*sameer
```

all of these are potentially valid:

```text
a... → asameer
b... → bsameer
c... → customersameer
j... → johnsameer
s... → sameer
x... → xsameer
```

Therefore, the suffix constraint provides weak pruning information near the root of a prefix-oriented dictionary.

---

### 3. Lucene Uses an Automaton

Lucene does not need to implement a leading wildcard as:

```text
for every term:
    if term.endsWith("sameer"):
        match
```

Conceptually, Lucene compiles the wildcard pattern into an automaton representing:

```text
ANYTHING*
    ↓
s → a → m → e → e → r
    ↓
 ACCEPT
```

Its language is:

```text
L = { all terms ending exactly in "sameer" }
```

Lucene can then intersect this automaton with the term dictionary using automaton-aware term enumeration.

Conceptually:

```text
*sameer
   ↓
WildcardQuery
   ↓
compile automaton
   ↓
automaton ∩ term dictionary
   ↓
matching terms
   ↓
postings
   ↓
matching documents
```

This is more precise than thinking of the operation as a naive scan of every term.

---

### 4. Why the Automaton Does Not Solve the Fundamental Problem

The automaton knows that the term must eventually end in:

```text
sameer
```

but near the beginning of a candidate term, almost anything remains possible.

For example, after consuming:

```text
john
```

the term could still become:

```text
johnsameer
```

Similarly:

```text
customer
```

could become:

```text
customersameer
```

Therefore, many dictionary branches remain potentially compatible with the automaton.

Contrast this with:

```text
sameer*
```

where the first character already eliminates every root branch except `s`.

Thus the automaton helps Lucene perform intelligent dictionary intersection, but a leading wildcard provides much weaker **early pruning** than a fixed-prefix query.

---

### 5. Toy Example

Suppose the dictionary contains:

```text
analytics
anderson
backend
backendengineer
customersameer
database
engineer
foosameer
johnsameer
khan
performance
performanceengineer
sameer
sameera
sameerengineer
smith
systems
systemsameer
xsameer
```

For:

```text
*sameer
```

the matching dictionary terms are:

```text
customersameer
foosameer
johnsameer
sameer
systemsameer
xsameer
```

Notice that these terms are scattered across forward lexicographic term space.

Suppose their postings are:

```text
customersameer → [D2, D17]
foosameer      → [D8]
johnsameer     → [D3, D9, D21]
sameer         → [D1, D7]
systemsameer   → [D4, D12]
xsameer        → [D14]
```

Then:

```text
M = 6 matching terms
P = 11 postings
```

The query may still require substantial dictionary exploration to **discover** those six terms.

---

### 6. Term Discovery vs Document Discovery

Leading wildcard performance becomes easier to understand when these operations are separated.

#### Term discovery

Determine which dictionary terms satisfy:

```text
*sameer
```

This is where the leading wildcard can be expensive.

#### Document discovery

Once a matching term has been found:

```text
johnsameer
```

Lucene can access its postings:

```text
johnsameer → [D3, D9, D21]
```

These are separate costs.

Therefore:

```text
small result set
    ≠
cheap query
```

A query could return only a handful of documents while still requiring broad dictionary exploration.

---

### 7. Cost Model

Let:

```text
V = total number of unique indexed terms
E = amount of dictionary structure explored
M = number of matching terms
P = total postings processed for matching terms
```

A useful conceptual model is:

```text
T_leadingWildcard
≈
T_automaton
+
T_dictionaryExploration
+
T_matchingTermEnumeration
+
T_postings
```

or abstractly:

```text
T_leadingWildcard
≈
O(A) + O(E) + O(M) + O(P)
```

The important variable is often:

```text
E
```

For a selective prefix query:

```text
sameer*
```

the fixed prefix allows Lucene to navigate toward a narrow term-space region, so:

```text
E << V
```

For:

```text
*sameer
```

there is no useful fixed leading prefix.

Therefore, dictionary exploration can become much broader:

```text
E → substantial fraction of term space
```

in an unfavorable case.

This does **not** imply that Lucene literally compares every indexed term character-by-character.

It means the lack of a fixed prefix greatly reduces the term dictionary's pruning power.

---

### 8. Matching-Term Count Is Not Term-Discovery Cost

Suppose:

```text
V = 20,000,000 unique terms
M = 6 matching terms
P = 11 postings
```

Even though:

```text
M << V
P << V
```

the query can still be expensive because Lucene must first **discover where those six terms exist**.

Therefore:

```text
result count ≠ query cost
```

and more specifically:

```text
matching-term count ≠ term-discovery cost
```

This is one of the central performance characteristics of leading wildcard queries.

---

### 9. Prefix vs Leading Wildcard

Conceptually:

```text
sameer*

known prefix
    ↓
strong term-space positioning
    ↓
narrow dictionary exploration
    ↓
matching terms
    ↓
postings
```

Whereas:

```text
*sameer

known suffix
    ↓
no useful forward-prefix position
    ↓
broad dictionary exploration
    +
automaton intersection
    ↓
matching terms
    ↓
postings
```

The fundamental problem is not that Lucene cannot express a suffix constraint.

The wildcard automaton expresses that constraint naturally.

The problem is that the **forward term dictionary cannot use the suffix to efficiently position itself in term space**.

---

### 10. Reversed-Term Optimization

Suffix search can be transformed into prefix search by maintaining an alternative representation containing reversed terms.

For example:

```text
johnsameer      → reemasnhoj
customersameer  → reemasremotsuc
systemsameer    → reemasmetsys
xsameer         → reemasx
```

Then:

```text
*sameer
```

can conceptually become:

```text
reemas*
```

against the reversed representation.

Now all matching terms share the fixed prefix:

```text
reemas
```

so they occupy a narrow region in the reversed term dictionary.

Conceptually:

```text
*sameer
    ↓
reverse suffix
    ↓
reemas*
    ↓
prefix-oriented dictionary lookup
    ↓
narrow term-space exploration
```

This illustrates a broader systems principle:

> **Change the data representation so that the query aligns with the index's natural access pattern.**

The tradeoff is additional indexing work and storage in exchange for cheaper suffix-search reads.

---

### Key Takeaway

The fundamental difference between:

```text
sameer*
```

and:

```text
*sameer
```

is not primarily the number of results.

It is the amount of **navigation information available before dictionary exploration begins**.

```text
sameer*
→ known beginning
→ aligns with forward term ordering
→ strong early pruning
```

```text
*sameer
→ unknown beginning
→ suffix constraint points in the opposite direction
→ weak early pruning
→ potentially broad dictionary exploration
```

Lucene's automaton machinery makes leading-wildcard execution smarter than a naive full-term scan, but it cannot eliminate the fundamental mismatch between a **suffix-oriented query** and a **prefix-oriented term dictionary**.
