---
title: Elasticsearch
description: A one-page, system-design summary and technical overview of Elasticsearch.
---

# Elasticsearch

## The one-page summary

Elasticsearch is a distributed search engine. It makes text searchable by
relevance ("red running shoes"), filterable, and aggregatable. It is usually a
**copy** of data whose source of truth lives in another database.

### Example

An online store keeps products in PostgreSQL and copies them into Elasticsearch
for search:

```json
PUT /products/_doc/123
{
  "name": "Lightweight Red Running Shoe",
  "brand": "Stride",
  "price": 89.99,
  "in_stock": true
}
```

A shopper searches "red running shoes" under $100:

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must":   { "match": { "name": "red running shoes" } },
      "filter": [
        { "range": { "price": { "lte": 100 } } },
        { "term":  { "in_stock": true } }
      ]
    }
  },
  "aggs": { "brands": { "terms": { "field": "brand" } } }
}
```

- `match` finds documents containing the words and ranks them by relevance.
  "shoes" also matches "Shoe" because text is normalized.
- `filter` applies exact yes/no conditions without affecting the ranking.
- `aggs` returns counts per brand for the sidebar filters.

A SQL `LIKE '%red running shoes%'` can't rank results and has to scan every row.

### Developer Choices

- **Source of truth and sync.** Decide which database owns the data and how
  changes reach Elasticsearch (usually [CDC or an outbox](./change-data-capture-outbox.md)).
- **Mappings (field types).** `text` for searchable words, `keyword` for exact
  values you filter or sort on (brand, status), plus number, date, and geo types.
  Define them explicitly.
- **Analyzers.** How text is split into words: lowercasing, stemming
  ("running" → "run"), synonyms ("sneaker" = "shoe").
- **Shards and replicas.** An index is split into primary shards (fixed at
  creation) and copied to replicas (changeable anytime).
- **Freshness.** New documents become searchable after a refresh, about every
  1 second by default.

### Pros

1. **Great text search:** relevance ranking, typo tolerance, synonyms,
   highlighting.
2. **Search, filters, and aggregations in one query:** ideal for faceted search.
3. **Scales out:** shards spread data; replicas add read capacity.
4. **Also does** geo search, vector search, and log analytics (with Kibana).

### Cons

- **Not a source of truth:** no joins or multi-document transactions.
- **Indexing is expensive:** analyzing text uses a lot of CPU, memory, and disk.
- **Hard to change later:** changing a field type or shard count means
  reindexing.
- **Slight delay:** a new document is searchable after about a second, not
  immediately.

### Numbers to Know

- Aim for **10–50 GB per shard**.
- Documents become searchable after about **1 second** (the refresh interval).
- Paging with `from`/`size` stops at **10,000 results**; use `search_after` to go
  deeper.
- An index allows **1,000 fields** by default.

### Scaling Reads

- Add replicas; any copy of a shard can serve searches.
- Search fewer shards: query only the indexes or time ranges you need.
- Use `filter` for exact conditions; filters are cached.
- Avoid deep paging and heavy aggregations on hot paths.

### Scaling Writes

- Index in bulk batches with several workers.
- Increase the refresh interval during big imports.
- Index only fields you actually search, filter, or sort on.
- For logs and other time-based data, roll over to a new index each day or at a
  size limit.

## Overview

### How it works

- **Inverted index:** like the index at the back of a book. It maps each word to
  the documents containing it: `"red" → [123, 456]`, `"shoe" → [123, 789]`.
  Finding matches is a lookup, not a scan.
- **Shards:** each document goes to one primary shard (by hashing its ID) and is
  copied to that shard's replicas.
- **Indexing:** the primary shard analyzes the text, logs the operation for
  durability, and forwards it to the replicas. The document becomes searchable
  at the next refresh.
- **Searching:** the receiving node sends the query to one copy of every
  relevant shard, merges each shard's top results, then fetches the winning
  documents. A search is only as fast as its slowest shard.
- **Updates:** data files are immutable, so an update writes a new version and
  marks the old one deleted. Background merges clean up.

### When to choose Elasticsearch

Choose it for product search, document search, log and metrics exploration,
autocomplete, faceted filters, and geo or vector search.

Don't use it as the only database for payments, inventory, or anything that
needs transactions. Keep the truth in PostgreSQL or DynamoDB, stream changes
into Elasticsearch, and accept that search results can lag by a second or two.
If the index breaks, you can rebuild it from the source.

### Further reading

- [Elastic documentation: index fundamentals](https://www.elastic.co/docs/manage-data/data-store/index-basics)
- [Elastic documentation: reading and writing documents](https://www.elastic.co/docs/deploy-manage/distributed-architecture/reading-and-writing-documents)
- [Elastic documentation: text analysis](https://www.elastic.co/docs/manage-data/data-store/text-analysis)
- [Elastic documentation: size your shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)
