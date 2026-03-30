---
slug: search-systems
title: Search Systems — Inverted Indexes, Elasticsearch, and Autocomplete
readTime: 20 min
orderIndex: 1
premium: false
---

# Search Systems

Every non-trivial application needs search. Understanding how search works under the hood — from inverted indexes to relevance scoring — will help you design search features that are fast, relevant, and scalable.

---

## Why Regular Databases Can't Do Search Well

```sql
-- This works, but it's terrible:
SELECT * FROM articles WHERE body LIKE '%system design%';
```

**Problems:**
1. Full table scan — reads every row regardless of index
2. No relevance ranking — mentioned once = same rank as mentioned 100 times
3. No stemming — "designing" won't match "design"
4. No fuzzy matching — typos return nothing
5. Index can't be used for leading wildcards (`%word%`)

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
┌─────────────────┬──────────────────────────────────────────┐
│   Term          │  Postings List (doc_id: positions)        │
├─────────────────┼──────────────────────────────────────────┤
│ "design"        │  Doc1:[2], Doc2:[2], Doc3:[3]             │
│ "system"        │  Doc1:[0], Doc2:[0]                       │
│ "systems"       │  Doc3:[1]                                 │
│ "scale"         │  Doc2:[3]                                 │
│ "fun"           │  Doc1:[3]                                 │
│ "distributed"   │  Doc3:[0]                                 │
└─────────────────┴──────────────────────────────────────────┘

Query: "system design"
→ Docs containing "system": Doc1, Doc2
→ Docs containing "design": Doc1, Doc2, Doc3
→ Intersection: Doc1, Doc2  ← Final results
```

Lookup is O(1) per term + merging postings lists. Dramatically faster than a full table scan.

---

## Text Processing Pipeline

Raw text goes through a pipeline before indexing (and again at query time):

```
Raw text: "The QUICK brown foxes jumped over LAZY dogs"
               │
               ▼
         1. Tokenization
               │   ["The","QUICK","brown","foxes","jumped","over","LAZY","dogs"]
               ▼
         2. Lowercasing
               │   ["the","quick","brown","foxes","jumped","over","lazy","dogs"]
               ▼
         3. Stop word removal
               │   remove: "the", "over"
               │   ["quick","brown","foxes","jumped","lazy","dogs"]
               ▼
         4. Stemming / Lemmatization
               │   foxes→fox, jumped→jump
               │   ["quick","brown","fox","jump","lazy","dog"]
               ▼
         5. Index tokens
               │   quick→[doc1], brown→[doc1], fox→[doc1], ...
```

### Stemming vs Lemmatization

| Approach | "running" | "better" | Speed | Accuracy |
|----------|-----------|----------|-------|----------|
| Stemming (Porter) | "run" | "better" | Fast | Good enough |
| Lemmatization | "run" | "good" | Slower | More accurate |

Both ensure "run", "running", and "runs" find the same documents.

---

## Relevance Scoring: TF-IDF and BM25

Not all matches are equal. A document mentioning "system design" 50 times is more relevant than one that mentions it once.

### TF-IDF (Term Frequency – Inverse Document Frequency)

```
TF (Term Frequency):
  How often does the term appear in this document?
  TF("design", Doc1) = 1/4 = 0.25  (1 occurrence in a 4-word doc)

IDF (Inverse Document Frequency):
  How rare is the term across all documents?
  IDF("design") = log(3 docs / 3 docs containing "design") = log(1) = 0
  IDF("fun")    = log(3 docs / 1 doc containing "fun")    = log(3) = 1.1

TF-IDF = TF × IDF
  "design" in Doc1 = 0.25 × 0   = 0.0   (common word → low value)
  "fun" in Doc1    = 0.25 × 1.1 = 0.28  (rare word → higher value)
```

**Intuition:** Rare words that appear frequently in a document are the strongest relevance signals.

### BM25 (Modern Standard)

BM25 (Best Match 25) improves on TF-IDF:
- **Length normalization** — long documents don't win just for mentioning a term more
- **Term frequency saturation** — 10 occurrences isn't 10x better than 1
- Standard in Elasticsearch, Solr, Lucene

---

## Elasticsearch Architecture

Elasticsearch is the most widely deployed search engine. It wraps Apache Lucene with a distributed, REST API layer.

```
┌─────────────────────────────────────────────────────────────┐
│                   Elasticsearch Cluster                     │
│                                                             │
│  Index: "articles"                                          │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Shard 0 (primary)     Shard 1 (primary)           │     │
│  │  + replica on Node B   + replica on Node A         │     │
│  │                                                    │     │
│  │  Shard 2 (primary)                                 │     │
│  │  + replica on Node A                               │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  Node A          Node B          Node C (coordinator)       │
└─────────────────────────────────────────────────────────────┘
```

**Key concepts:**
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
   c. Periodically flushes buffer to on-disk Lucene segment (refresh interval: 1s)
4. Replicates to replica shards
5. Returns success when primary + replicas acknowledge

Important: Documents appear in search ~1 second after indexing.
Elasticsearch is "near real-time", not real-time.
```

---

## Read Path (Search: Scatter-Gather)

```
Query: "system design"

1. Client sends query to any node (coordinator)
2. Coordinator broadcasts query to one shard copy per group (scatter)
   → Shard 0, Shard 1, Shard 2 each search locally in parallel
3. Each shard returns its local top-K results with scores
4. Coordinator merges and re-ranks all results (gather)
5. Coordinator fetches full documents for the final top results
6. Returns ranked list to client
```

---

## Autocomplete / Typeahead

As users type, show real-time suggestions.

### Prefix Matching with a Trie

```
Trie for: ["system", "sys", "search", "scale"]

         (root)
        /      \
       s        ...
      / \
    sy   se
    |      \
   sys     sea
    |         \
   syst       sear
    |            \
  syste         searc
    |               \
  system            search ✓

Query prefix "sys" → traverse to "sys" node → return: ["sys", "system"]
```

### N-gram Index (Substring Search)

```
"search" → trigrams: ["sea", "ear", "arc", "rch"]

Query "arc" → finds "search" even without typing from the start
```

Edge n-grams (prefix-only) are common for autocomplete. Full n-grams enable substring matching.

---

## Fuzzy Matching (Typo Tolerance)

"sysetm" should still find "system".

**Edit distance (Levenshtein):** Minimum single-character edits (insert, delete, substitute) to transform one string into another.

```
"sysetm" → "system":  edit distance = 2
```

In Elasticsearch:
```json
{
  "query": {
    "fuzzy": {
      "title": { "value": "sysetm", "fuzziness": 2 }
    }
  }
}
```

---

## Relevance Beyond Text — Modern Ranking Signals

| Signal Type | Examples |
|-------------|---------|
| Text relevance | BM25 score, title match bonus, exact phrase match |
| Popularity | Click-through rate, view count, star count |
| Freshness | Recent documents ranked higher (news) |
| Personalization | User's past searches and clicks |
| Authority | PageRank, domain authority, backlinks |
| Semantic similarity | Vector embeddings (neural/semantic search) |

**Learning to Rank (LTR):** Train an ML model on user click data to optimally combine these signals. Used by Bing, LinkedIn, and Airbnb.

---

## SimHash for Near-Duplicate Detection

Search engines must avoid indexing duplicate content. SimHash produces a fingerprint where similar documents produce **similar** hashes:

```
Document A: "The quick brown fox"
Document B: "The quick brown dog"   ← nearly identical

MD5(A)     = 9a... ≠ MD5(B) = 4f...    (completely different)
SimHash(A) = 10110110
SimHash(B) = 10110100                   ← 1 bit different!

Hamming distance = 1  →  near-duplicates detected
```

---

## Key Takeaways

1. **Inverted indexes are why search is fast** — O(1) term lookup vs O(n) full table scan
2. **Text processing normalizes documents and queries** — stemming ensures "running" finds "run"
3. **BM25 is the modern relevance standard** — better than TF-IDF, used in Elasticsearch/Lucene
4. **Elasticsearch shards enable parallel search** — scatter-gather across shards, merge at coordinator
5. **Autocomplete uses tries or edge n-gram indexes** — prefix matching is simple and fast
6. **Fuzzy matching handles typos** — Levenshtein distance with configurable fuzziness level
7. **Modern search blends text + popularity + ML signals** — pure text relevance is never enough
