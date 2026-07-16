---
layout: post
title: Overview
description: Learn more about Zulia
image: assets/images/overview.jpg
---

## Zulia
### Distributed Lucene with deep object searching

Zulia is a real-time distributed search and storage system. Zulia is designed to scale both vertically and horizontally across servers.

### Zulia is:
* Realtime
* Distributed
* Pure Java with [rich Java client]({% post_url 2022-6-13-java %})
* Open Source
* Based on Lucene 10.5.0

### Zulia supports:
* [Searching multiple indexes (including wildcard patterns) with a single query]({% post_url 2022-6-13-java %}#search-multiple-indexes)
* [Storing associated files with the documents (images, pdfs, ...)]({% post_url 2022-6-13-java %}#storing-associated-documents)
* [Sorting]({% post_url 2022-6-13-java %}#sorting)
* [Facet Counts]({% post_url 2022-6-13-java %}#count-facets)
* [Statistics]({% post_url 2022-6-13-java %}#numeric-stat) and [Facet Statistics]({% post_url 2022-6-13-java %}#stat-facet)
* [Vector and Hybrid Search (BM25 + vector)]({% post_url 2022-6-13-java %}#vector-queries)
* [Vector Quantization (INT8, INT4, BBQ) with full-precision rescoring]({% post_url 2022-6-13-java %}#vector-quantization)
* [More-Like-This Queries (lexical, vector, and hybrid)]({% post_url 2022-6-13-java %}#more-like-this-queries)
* [Geo Point Search]({% post_url 2022-6-13-java %}#geo-point-fields)
* [Multi-index Aliases with a designated Write Index]({% post_url 2022-6-13-java %}#index-aliases)
* [Transient Indexes with lazy loading and eviction]({% post_url 2022-6-13-java %}#transient-indexes)
* [Lucene Segment Replication across nodes]({% post_url 2022-6-14-install %}#replication)
* [Rich Query Syntax]({% post_url 2022-6-10-query-syntax %})

### Learn more
* [Install]({% post_url 2022-6-14-install %})
* [Sharding and Replication]({% post_url 2022-6-13-sharding %})
* [Command Line Client]({% post_url 2022-6-11-cli %})
* [Java Client]({% post_url 2022-6-13-java %})
* [Rest Service]({% post_url 2022-6-12-rest %})
* [Query Syntax]({% post_url 2022-6-10-query-syntax %})
* [Testing]({% post_url 2022-6-9-testing %})
* [Data Library]({% post_url 2022-6-8-data-library %})


### Requirements
* Java 25
* MongoDB (for cluster mode)

### Latest Release
* Version 5.3.0 - [Download](https://github.com/zuliaio/zuliasearch/releases)
