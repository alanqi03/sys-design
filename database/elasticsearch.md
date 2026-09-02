---
title: Elasticsearch
description: A one-page, system-design summary and technical overview of Elasticsearch.
---

# Elasticsearch

## The one-page summary

Elasticsearch is a distributed search and analytics engine built on Apache
Lucene. It turns JSON documents into structures optimized for full-text search,
filtering, relevance ranking, aggregations, geospatial queries, and vector
similarity—usually as a search projection of data owned by another system.

### Developer Choices

- **System of record and synchronization.** Decide which database owns the data
  and how changes reach Elasticsearch, such as an outbox plus CDC, a stream, or
  periodic reindexing. Design for duplicate, delayed, and out-of-order events.
- **Mappings and field types.** Use `text` for analyzed natural language,
  `keyword` for exact filters/sorts/aggregations, and explicit numeric, date,
  geo, object, nested, or vector types. Prefer explicit production mappings so
  one malformed document cannot define the wrong type.
- **Analysis and relevance.** Choose tokenizers, normalization, stemming,
  synonyms, language analyzers, field boosts, and ranking behavior; test them
  against judged queries rather than relying only on defaults.
- **Indices, shards, and routing.** Decide index boundaries, primary-shard count,
  replica count, and whether tenant or entity routing can limit searches to
  fewer shards. Primary-shard count is fixed when an index is created.
- **Freshness and durability.** Choose an acceptable refresh delay and whether
  writes must wait for search visibility. Also choose how many active shard
  copies must exist before accepting writes and how snapshots will be managed.
- **Lifecycle.** Define rollover, retention, hot/warm/cold tiers, deletion, and
  reindexing for mapping changes before continuously growing indices become an
  operational problem.

### Pros

1. **Excellent search quality.** Inverted indices, analyzers, BM25 ranking,
   phrase/prefix/fuzzy queries, highlighting, and synonyms support rich lexical
   search.
2. **Flexible retrieval and analytics.** Filters, faceting, aggregations,
   geospatial queries, vector search, and hybrid ranking share one engine.
3. **Horizontal search scale.** Primary shards distribute documents and query
   work; replicas add redundancy and read capacity.
4. **Mature ecosystem.** Kibana, ingest pipelines, connectors, language clients,
   monitoring, lifecycle automation, and Elastic Cloud reduce integration work.

### Cons

- **It is not a relational transaction system.** Elasticsearch has no joins or
  general multi-document transactions and is usually a poor source of truth for
  business invariants.
- **Indexing is resource intensive.** Text analysis, multiple field structures,
  replicas, refreshes, and segment merging amplify CPU, memory, disk, and I/O.
- **Mappings and shards require foresight.** Many mapping changes require
  reindexing, while too many tiny shards or a few huge shards hurt stability and
  recovery.
- **Search is near-real-time.** An acknowledged write is durable but may not be
  visible to search until refresh, so read-after-write workflows need an
  explicit strategy.

### Numbers to Know

- Aim for roughly **10–50 GB per shard** and fewer than **200 million documents
  per shard**, then benchmark recovery and query latency with real data.
- Active indices in the versioned Elastic Stack refresh about every **1 second**
  by default; Elastic Cloud Serverless defaults to **5 seconds**.
- The default `index.max_result_window` permits `from`/`size` pagination through
  **10,000 hits**; use `search_after` with a point in time for deeper traversal.
- The default mapping limit is **1,000 fields per index** to prevent mapping
  explosion and excessive memory use.
- Keep a bulk request below roughly **a few tens of megabytes** and discover its
  optimal document count by benchmarking progressively larger batches.
- There is no useful generic search QPS: a cached term lookup and a cross-shard
  aggregation or vector query can differ by orders of magnitude.

### Scaling Reads

- **Add replicas.** Elasticsearch can serve a search from either a primary or a
  replica shard, increasing parallel read capacity and resilience.
- **Use enough—but not too many—primary shards.** Shards enable parallelism, but
  every shard adds heap, coordination, and per-query work.
- **Search fewer shards.** Narrow index patterns, time ranges, and custom routing
  prevent every request from fanning out across the whole cluster.
- **Make queries cheaper.** Prefer filters for exact constraints, map sortable
  fields with doc values, cap expensive aggregations, avoid scripts on hot paths,
  and use `search_after` instead of deep offset pagination.
- **Separate workloads.** Dedicated coordinating nodes, content/search tiers,
  cached repeated queries, and asynchronous search can protect interactive
  latency from heavy analytics.

### Scaling Writes

- **Use bulk indexing with several workers.** Increase batch size and concurrency
  until throughput plateaus, then back off with jitter after HTTP `429` responses.
- **Distribute routing.** Enough primary shards and well-distributed document IDs
  let multiple data nodes index concurrently; avoid routing one hot tenant to a
  single shard.
- **Relax freshness when possible.** A longer refresh interval creates fewer,
  larger segments and improves ingestion throughput at the cost of search delay.
- **Reduce index amplification.** Index only fields that need search, sort, or
  aggregation; limit analyzers, multi-fields, stored values, and replicas to
  what the workflow requires.
- **Roll over growing indices.** Data streams and lifecycle policies create new
  write indices based on size, age, or document count and move older data to
  cheaper tiers.

## Overview

### Notes

- Treat Elasticsearch as a **rebuildable derived index** when possible. Keep the
  authoritative record and an ordered change stream elsewhere.
- A document update writes a new Lucene document version and later deletes the
  old one during segment merging; frequent partial updates are not cheap in-place
  mutations.
- Use `_seq_no` and `_primary_term` for optimistic concurrency control so a stale
  client cannot overwrite a newer document version.
- Write acknowledgement and search visibility are different events. Use
  `refresh=wait_for` only when a workflow truly needs immediate search-after-write
  behavior; forcing `refresh=true` repeatedly creates costly tiny segments.

### How It Works

#### Indices, shards, and Lucene segments

An Elasticsearch index is a logical collection of JSON documents with mappings
and settings. It is divided into **primary shards**, and every primary may have
one or more **replica shards** on other nodes. Each shard is an independent
Lucene index made of immutable **segments**.

Elasticsearch routes a document to one primary shard, normally by hashing its
document ID. Primary-shard count is fixed at index creation because changing it
would change this routing. Replica count is dynamic. Cluster-manager-eligible
nodes maintain cluster metadata and allocation decisions, while data nodes store
and execute work on shards; any node receiving a request acts as its coordinator.

#### What happens while indexing

1. An ingest pipeline may parse, enrich, normalize, or reject the incoming JSON
   document before storage.
2. Elasticsearch resolves the target index or data stream, validates the mapping,
   and hashes the ID or custom routing value to a replication group.
3. The coordinating node forwards the operation to that group's primary shard.
   The primary validates and indexes it locally, assigns a sequence number, and
   appends the operation to the translog.
4. The primary forwards the operation in parallel to every current in-sync
   replica. It acknowledges the client after those copies complete, subject to
   the active-shard requirements and durability settings.
5. Index-time analyzers transform `text` into tokens. Lucene builds an inverted
   index from terms to documents, while structures such as doc values support
   sorting and aggregations and vector fields support similarity search.
6. A refresh opens recently written Lucene segments for searching. A later flush
   creates a durable Lucene commit point and allows older translog generations to
   be removed.

With the default `request` translog durability, Elasticsearch fsyncs the translog
after each index, delete, update, or bulk request. An acknowledged document can
therefore survive recovery before it has become visible to search.

#### What happens during a search

1. The node receiving the request becomes the coordinating node and resolves the
   target indices, routing, aliases, filters, and relevant shards.
2. In the **scatter/query phase**, it selects one active copy of every relevant
   shard—primary or replica—and sends each a shard-level query.
3. Each shard analyzes full-text input, searches its local immutable segments,
   computes scores or aggregations, and returns its best candidate document IDs
   and partial results.
4. The coordinator merges shard rankings and aggregation states into a global
   result. Distributed relevance statistics and per-shard candidate limits can
   affect the final ordering.
5. In the **fetch phase**, the coordinator asks the owning shards for `_source`
   and requested fields for the winning document IDs, then returns the response.

Search latency is bounded by the slowest participating shard plus coordination
and reduction work. A request touching hundreds of shards can therefore be much
more expensive than the same query routed to one shard.

#### Refreshes, updates, deletes, and merges

Lucene segments are immutable. A refresh creates searchable segment structures
for recent operations without performing a full durable commit. An update
indexes a replacement document and marks the prior version deleted; a delete
adds a deletion marker. Background merges combine smaller segments, discard
obsolete documents, and reclaim space.

Frequent refreshes make new documents searchable sooner but create more small
segments and more merge work. Large ingestion jobs often lengthen or temporarily
disable the refresh interval, restore it after loading, and allow merging to
catch up.

#### Replicas, failure, and recovery

Each primary shard and its replicas form a replication group. The primary is the
entry point for writes and keeps every in-sync copy updated. Searches can use any
active copy, and adaptive replica selection favors copies with better recent
latency and queue conditions.

If a primary fails, an in-sync replica can be promoted. Elasticsearch allocates
and recovers missing copies while respecting node, zone, tier, and disk rules.
Replicas protect against node loss, but they are not backups: snapshots to an
external repository are needed for accidental deletion, corruption, or
cluster-wide disaster recovery.

### Features

- Full-text queries, BM25 relevance, phrase and proximity matching, fuzziness,
  autocomplete patterns, synonyms, highlighting, and per-field boosting
- Exact filters, compound Query DSL, runtime fields, sorting, collapsing, and
  point-in-time searches with `search_after`
- Bucket, metric, pipeline, matrix, and geospatial aggregations for facets and
  analytics
- Dense and sparse vector fields, approximate k-nearest-neighbor search,
  semantic search, hybrid retrieval, and reciprocal-rank fusion
- `geo_point` and `geo_shape` indexing, distance queries, spatial relationships,
  sorting, and aggregations
- Explicit and dynamic mappings, custom analyzers, normalizers, multi-fields,
  aliases, index templates, and ingest pipelines
- Primary and replica shards, allocation awareness, automatic failover,
  cross-cluster search, and cross-cluster replication
- Bulk, multi-search, asynchronous search, tasks, reindex, update-by-query, and
  delete-by-query APIs
- Data streams, rollover, index/data-stream lifecycle management, hot/warm/cold
  tiers, searchable snapshots, and downsampling
- Security, API keys, role-based access, field/document-level security, audit
  logging, monitoring, and Kibana integration, depending on distribution and
  license

### When to choose Elasticsearch

Choose Elasticsearch for product or document search, log and observability
exploration, security analytics, catalog discovery, geospatial retrieval,
faceting, and lexical/vector hybrid search. It is particularly useful when users
need ranked relevance or flexible filters across text and structured metadata.

Do not choose it as the only database for payments, inventory, relational
workflows, or other state governed by cross-record invariants. A common design
commits authoritative state to PostgreSQL, DynamoDB, or another transactional
store, publishes changes through an outbox or CDC stream, and updates
Elasticsearch asynchronously. The search UI then tolerates brief staleness and
can rebuild the index from the source of truth.

### Sources and further reading

- [Elastic documentation: index fundamentals](https://www.elastic.co/docs/manage-data/data-store/index-basics)
- [Elastic documentation: reading and writing documents](https://www.elastic.co/docs/deploy-manage/distributed-architecture/reading-and-writing-documents)
- [Elastic documentation: text analysis](https://www.elastic.co/docs/manage-data/data-store/text-analysis)
- [Elastic documentation: search shard routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-shard-routing.html)
- [Elastic documentation: size your shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)
- [Elastic documentation: tune indexing speed](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/indexing-speed)
- [Elastic documentation: refresh behavior](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter)
- [Elastic documentation: optimistic concurrency control](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/optimistic-concurrency-control)
- [Elastic documentation: mapping limits](https://www.elastic.co/docs/reference/elasticsearch/index-settings/mapping-limit)
- [Elastic documentation: lifecycle and rollover](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/rollover)
- [Elastic documentation: hybrid search](https://www.elastic.co/docs/solutions/search/hybrid-search)
