# Search Systems

Every non-trivial application needs search. Understanding how search works under the hood — from inverted indexes to relevance scoring — will help you design search features that are fast, relevant, and scalable.

---

## Why Regular Databases Can't Do Search Well

```sql
-- This works, but it's terrible:
SELECT * FROM articles WHERE body LIKE '%system design%';

Problems:
1. Full table scan — reads every row
2. No relevance ranking — "system design" mentioned once = same as 100 times
3. No stemming — "designing" won't match "design"
4. No fuzzy matching — typos return nothing
5. Index can't be used for leading wildcards (%word%)
```

Full-text search requires purpose-built data structures.

---

## The Inverted Index

The inverted index is the core data structure of every search engine. It maps words to the documents that contain them — the inverse of a document storing its own words.

### Building an Inverted Index

```
Documents:
  Doc 1: "system design is fun"
  Doc 2: "system design at scale"
  Doc 3: "distributed systems design"

Inverted Index:
┌────────────┬────────────────────────────────────────┐
│   Term     │  Postings List (doc_id: positions)      │
├────────────┼────────────────────────────────────────┤
│ "design"   │  Doc1:[2], Doc2:[2], Doc3:[3]           │
│ "system"   │  Doc1:[0], Doc2:[0]                     │
│ "systems"  │  Doc3:[1]                               │
│ "scale"    │  Doc2:[3]                               │
│ "fun"      │  Doc1:[3]                               │
│ "distributed"│ Doc3:[0]                              │
└────────────┴────────────────────────────────────────┘

Query: "system design"
→ Fetch Doc1, Doc2 (contain "system") ∩ Doc1, Doc2, Doc3 (contain "design")
→ Result: Doc1, Doc2 (both contain "system" AND "design")
```

Lookup is O(1) per term + merging postings lists. Dramatically faster than full table scan.

---

## Text Processing Pipeline

Raw text goes through a pipeline before indexing (and again at query time):

```
Raw text: "The QUICK brown foxes jumped over LAZY dogs"
               │
               ▼
         1. Tokenization
               │   ["The", "QUICK", "brown", "foxes", "jumped", "over", "LAZY", "dogs"]
               ▼
         2. Lowercasing
               │   ["the", "quick", "brown", "foxes", "jumped", "over", "lazy", "dogs"]
               ▼
         3. Stop word removal
               │   (remove: "the", "over")
               │   ["quick", "brown", "foxes", "jumped", "lazy", "dogs"]
               ▼
         4. Stemming / Lemmatization
               │   foxes→fox, jumped→jump, lazy→lazi (stemming)
               │   ["quick", "brown", "fox", "jump", "lazi", "dog"]
               ▼
         5. Index tokens
               │   quick→[doc1], brown→[doc1], fox→[doc1], ...
```

### Stemming vs Lemmatization

| Approach | "running" | "better" | Speed |
|----------|-----------|----------|-------|
| Stemming (Porter) | "run" | "better" | Fast |
| Lemmatization | "run" | "good" | Slower |

Stemming is a heuristic (chop suffixes). Lemmatization uses a dictionary to find the true base form. Both make "run", "running", "runs" find the same documents.

---

## Relevance Scoring: TF-IDF

Not all matches are equal. A document mentioning "system design" 50 times is more relevant than one that mentions it once.

### TF-IDF (Term Frequency – Inverse Document Frequency)

```
TF (Term Frequency):
  How often does the term appear in this document?
  TF("design", Doc1) = 1/4 = 0.25  (1 occurrence in 4-word doc)

IDF (Inverse Document Frequency):
  How rare is the term across all documents?
  IDF("design") = log(3 docs / 3 docs containing "design") = log(1) = 0
  IDF("fun")    = log(3 docs / 1 doc containing "fun")    = log(3) = 1.1

TF-IDF = TF × IDF
  "design" in Doc1 = 0.25 × 0 = 0.0  (common word, low value)
  "fun" in Doc1    = 0.25 × 1.1 = 0.28 (rare word, higher value)
```

**Intuition:** Rare words that appear frequently in a document are the best signals of relevance.

### BM25 (Better Practical Formula)

Most modern search engines use BM25 (Best Match 25), which improves on TF-IDF:

- Handles document length normalization (a long doc shouldn't win just for mentioning a term more)
- Saturating term frequency (10 occurrences isn't 10x better than 1)
- Standard in Elasticsearch, Solr, Lucene

---

## Elasticsearch Architecture

Elasticsearch is the most widely deployed search engine. It wraps Apache Lucene with a distributed, REST API layer.

```
┌─────────────────────────────────────────────────────────────┐
│                   Elasticsearch Cluster                     │
│                                                             │
│  ┌──────────────────────────────────────────────────┐       │
│  │                     Index                        │       │
│  │   "articles"                                     │       │
│  │                                                  │       │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────┐ │       │
│  │  │  Shard 0     │  │  Shard 1     │  │ Shard 2│ │       │
│  │  │  (primary)   │  │  (primary)   │  │(primary│ │       │
│  │  │  + 1 replica │  │  + 1 replica │  │+1 rep) │ │       │
│  │  │  on Node B   │  │  on Node A   │  │on NodeA│ │       │
│  │  └──────────────┘  └──────────────┘  └────────┘ │       │
│  └──────────────────────────────────────────────────┘       │
│                                                             │
│  Node A          Node B           Node C (coordinator)      │
└─────────────────────────────────────────────────────────────┘
```

Key concepts:
- **Index** — collection of documents (like a database table)
- **Shard** — a Lucene index; the unit of parallelism and distribution
- **Replica** — copy of a shard for high availability and read throughput
- **Node** — single Elasticsearch server instance

---

## Write Path (Indexing)

```
1. Client sends document via REST API
2. Coordinating node hashes document ID → determines primary shard
3. Primary shard:
   a. Writes to in-memory buffer (fast)
   b. Appends to translog (durability)
   c. Periodically flushes buffer to on-disk Lucene segment (refresh)
4. Replicates to replica shards
5. Returns success when primary + replicas acknowledge

Refresh interval: 1 second (default)
→ Documents appear in search ~1 second after indexing
→ Elasticsearch is "near real-time", not real-time
```

---

## Read Path (Search: Scatter-Gather)

```
Query: "system design"

1. Client sends query to any node (coordinator)
2. Coordinator broadcasts query to one shard copy per shard group (scatter)
   → Shard 0, Shard 1, Shard 2 each search locally
3. Each shard returns its top-K results
4. Coordinator merges and re-ranks all results (gather)
5. Coordinator fetches full documents for top results
6. Returns final ranked list to client
```

This is the **scatter-gather** pattern — broadcast to all shards, merge results.

---

## Autocomplete / Typeahead

As users type, show suggestions in real time.

### Prefix Matching with a Trie

```
Trie for: ["system", "sys", "search", "scale"]

         (root)
        /      \
       s         ...
      / \
    sy    se
    |      \
   sys      sea
    |         \
  syst        sear
    |            \
  syste          searc
    |               \
  system            search
  (end)             (end)
```

Query prefix "sys" → traverse to "sys" node → all words below → ["sys", "system"]

### N-gram Index

Break words into character n-grams and index each:

```
"search" → ["sea", "ear", "arc", "rch"]  (trigrams)

Query "arc" → finds "search" even without full prefix
```

**Use case:** Edge n-gram (prefix-only) is common for autocomplete. Full n-grams enable substring search.

---

## Fuzzy Matching

Typo-tolerant search: "sysetm" should still find "system".

**Edit distance (Levenshtein):** Minimum number of single-character edits (insert, delete, substitute) to transform one word into another.

```
"sysetm" → "system":  edit distance = 2 (swap 'e' and 't', swap 'm' and 'e')
```

Elasticsearch supports fuzzy queries:
```json
{
  "query": {
    "fuzzy": { "title": { "value": "sysetm", "fuzziness": 2 } }
  }
}
```

---

## Relevance Beyond Text: Ranking Signals

Modern search uses much more than TF-IDF/BM25:

| Signal Type | Examples |
|-------------|---------|
| Text relevance | BM25 score, title match bonus, exact phrase match |
| Popularity | Click-through rate, view count, star count |
| Freshness | Recent documents ranked higher (news search) |
| Personalization | User's past searches and clicks |
| Authority | PageRank, domain authority, backlinks |
| Semantic similarity | Vector embeddings (neural search) |

**Learning to Rank (LTR):** Train an ML model to combine signals optimally based on user click data.

---

## Search Architecture for a Learning Platform

```
┌──────────────┐      search query       ┌──────────────────┐
│   Browser    │───────────────────────▶ │  Search API      │
│              │◀─── ranked results ───  │  (Express/FastAPI)│
└──────────────┘                         └──────┬───────────┘
                                                │
                                   ┌────────────▼────────────┐
                                   │    Elasticsearch        │
                                   │   articles index        │
                                   │   courses index         │
                                   └────────────┬────────────┘
                                                │ indexes
                                   ┌────────────▼────────────┐
                                   │  Indexing Pipeline      │
                                   │  (on content publish)   │
                                   └────────────┬────────────┘
                                                │ reads
                                   ┌────────────▼────────────┐
                                   │   Content Database      │
                                   │   (PostgreSQL)          │
                                   └─────────────────────────┘
```

---

## Key Takeaways

1. **Inverted indexes are why search is fast** — O(1) term lookup vs O(n) full table scan
2. **Text processing normalizes documents and queries** — stemming ensures "running" finds "run"
3. **BM25 is the standard relevance algorithm** — better than TF-IDF, used in Elasticsearch/Lucene
4. **Elasticsearch shards enable parallel search** — scatter-gather across shards, merge on coordinator
5. **Autocomplete uses tries or n-gram indexes** — prefix matching is simple and fast
6. **Fuzzy matching handles typos** — Levenshtein distance with configurable fuzziness
7. **Modern search blends many signals** — text relevance + popularity + freshness + ML ranking

---

*Previous: [Microservices vs Monolith ←](07-microservices-vs-monolith.md) · Next: [Blob Storage and Object Storage →](09-blob-storage.md)*
