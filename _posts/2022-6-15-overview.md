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
* Pure Java with [rich Java client](Java-Client)
* Open Source
* Based on Lucene 10.4.0

### Zulia supports:
* [Searching multiple indexes with a single query](Java-Client#search-multiple-indexes)
* [Storing associated files with the documents (images, pdfs, ...)](Java-Client#storing-associated-documents)
* [Sorting](Java-Client#sorting)
* [Facet Counts](Java-Client#count-facets)
* [Statistics](Java-Client#numeric-stat) and [Facet Statistics](Java-Client#stat-facet)
* [Vector Search](Java-Client#vector-queries)
* [Geo Point Search](Java-Client#geo-point-fields)
* [Rich Query Syntax](Query-Syntax)

### Learn more
* [Install]({% post_url 2022-6-14-install %})
* [Command Line Client]({% post_url 2022-6-11-cli %})
* [Java Client]({% post_url 2022-6-13-java %})
* [Rest Service]({% post_url 2022-6-12-rest %})
* [Query Syntax]({% post_url 2022-6-10-query-syntax %})
* [Testing]({% post_url 2022-6-9-testing %})
* [Data Library]({% post_url 2022-6-8-data-library %})


### Requirements
* Java 21-25 (Tested on Java 21 and Java 25)
* MongoDB (for cluster mode)

### Latest Release
* Version 4.15.5 - [Download](https://github.com/zuliaio/zuliasearch/releases)
