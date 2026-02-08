---
layout: post
title: Testing
description: Validate search results with the Zulia test framework
---
# Zulia Test Framework

The `zuliatest` command is a YAML-driven testing framework for validating Zulia search results. It executes queries against a Zulia cluster and evaluates JavaScript expressions to assert expected results, producing a CSV report.

This is useful for search quality regression testing, CI/CD integration, and validating index data after migrations.

## Running Tests

```bash
zuliatest --testConfig /path/to/test_config.yaml --testOutput /path/to/results.csv
```

### Options
```
--testConfig  Full path to the test config YAML file (required)
--testOutput  Full path to the test output CSV file (required)
--showStack   Show stack traces on errors
--version     Show version info
```

### Exit Codes
* `0` - All tests passed
* `1` - One or more tests failed
* `9` - Configuration file not found or not readable

## YAML Configuration

The test configuration has four main sections: connections, indexes, searches, and tests.

### Connections

Define one or more Zulia server connections.

```yaml
connections:
  - name: local
    serverAddress: localhost
    port: 32191
  - name: production
    serverAddress: 10.0.0.50
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| name | yes | | Unique connection identifier |
| serverAddress | yes | | Hostname or IP address |
| port | no | 32191 | Zulia gRPC port |

### Indexes

Map logical index names to actual Zulia indexes on a connection.

```yaml
indexes:
  - name: pubs
    indexName: publications
    connection: local
```

| Field | Required | Description |
|-------|----------|-------------|
| name | yes | Logical name used in searches |
| indexName | yes | Actual Zulia index name |
| connection | yes | References a connection name |

### Searches

Define queries to execute. Each search produces a named result object accessible in test expressions.

```yaml
searches:
  - name: allDocs
    index: pubs
    queries:
      - q: "*:*"

  - name: cancerDocs
    index: pubs
    queries:
      - q: "cancer treatment"
        qf: [title, abstract]
        mm: 1
      - q: "year:[2020 TO *]"
        queryType: FILTER
    amount: 10
    documentFields: [title, authors, year]
    facets:
      - field: year
        topN: 10
    statFacets:
      - facetField: year
        numericField: citationCount
        topN: 10
    numStats:
      - numericField: year
        percentiles: [0.25, 0.5, 0.75]
        percentilePrecision: 0.01
```

#### Search Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| name | yes | | Unique identifier, used in test expressions |
| index | yes | | References an index name |
| queries | no | | List of query clauses |
| amount | no | 0 | Number of documents to return (0 = count only) |
| documentFields | no | | Specific fields to retrieve from documents |
| facets | no | | Count facet configurations |
| statFacets | no | | Statistical facet configurations |
| numStats | no | | Numeric field statistics |

#### Query Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| q | yes | | Query string (Lucene syntax) |
| queryType | no | SCORE_MUST | SCORE_MUST, SCORE_SHOULD, FILTER, or FILTER_NOT |
| qf | no | | Query fields (used when query has no explicit field) |
| mm | no | 0 | Minimum number of terms that must match |

#### Facet Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| field | yes | | Facet field name |
| topN | no | 0 | Number of top facet values to return |

#### Stat Facet Fields

| Field | Required | Description |
|-------|----------|-------------|
| facetField | yes | Field to facet on |
| numericField | yes | Numeric field for statistics |
| topN | no | Number of top facet values |

#### Numeric Stat Fields

| Field | Required | Description |
|-------|----------|-------------|
| numericField | yes | Numeric field name |
| percentiles | no | List of percentile points (0.0 to 1.0) |
| percentilePrecision | no | Precision for percentile calculation |

### Tests

JavaScript expressions that validate search results. Each expression must evaluate to `true` (PASS) or `false` (FAIL).

```yaml
tests:
  - name: hasResults
    expr: allDocs.count > 0

  - name: mostHaveTitles
    expr: cancerDocs.count > allDocs.count * 0.5

  - name: topYearRecent
    expr: cancerDocs.facet["year"][0].label == "2024"
```

### Logging

Optional flags for debugging.

```yaml
logSearches: true       # Log search queries as they execute
logSearchResults: true  # Log full search results as JSON
```

## Test Expression Reference

Each search result is available as a JavaScript object using the search's `name`. The following properties are available:

### count

Total hit count from the search.

```javascript
allDocs.count > 100000
cancerDocs.count < allDocs.count * 0.5
```

### doc

Array of returned documents (only populated when `amount` > 0). Supports nested field access.

```javascript
cancerDocs.doc[0].title == "Expected Title"
cancerDocs.doc[0]["authors"][0]["lastName"] == "Smith"
cancerDocs.doc.length == 10
```

### facet

Count facet results keyed by field name. Each entry has `label` (string) and `count` (long).

```javascript
cancerDocs.facet["year"][0].label == "2024"
cancerDocs.facet["year"][0].count > 50000
```

### statFacet

Statistical facet results keyed by `"facetField-numericField"`. Each entry has `label`, `docCount`, `allDocCount`, `valueCount`, `sum`, `max`, `min`, and optionally `percentiles`.

```javascript
cancerDocs.statFacet["year-citationCount"][0].label == "2024"
cancerDocs.statFacet["year-citationCount"][0].sum > 100000
cancerDocs.statFacet["year-citationCount"][0].docCount > 50000
```

### numStat

Numeric field statistics keyed by field name. Each entry has `docCount`, `allDocCount`, `valueCount`, `sum`, `max`, `min`, and optionally `percentiles`. Each percentile has `point` and `value`.

```javascript
// Average year
(cancerDocs.numStat["year"].sum / cancerDocs.numStat["year"].docCount) > 2010

// Percentiles
cancerDocs.numStat["year"].percentiles[0].value < 2000
cancerDocs.numStat["year"].percentiles[1].value > 2015
```

## CSV Output

The output CSV has two columns: `testId` and `result`.

```
testId,result
hasResults,PASS
mostHaveTitles,PASS
topYearRecent,FAIL
```

## Complete Example

```yaml
logSearches: false
logSearchResults: false

connections:
  - name: local
    serverAddress: localhost

indexes:
  - name: pubs
    indexName: publications
    connection: local

searches:
  - name: allPubs
    index: pubs
    queries:
      - q: "*:*"

  - name: recentCancer
    index: pubs
    queries:
      - q: "cancer"
        qf: [title, abstract]
      - q: "year:[2020 TO *]"
        queryType: FILTER
    amount: 5
    documentFields: [title, authors, year]
    facets:
      - field: year
        topN: 5
    numStats:
      - numericField: year
        percentiles: [0.25, 0.5, 0.75]

tests:
  - name: hasResults
    expr: allPubs.count > 0

  - name: cancerSubset
    expr: recentCancer.count < allPubs.count

  - name: hasDocuments
    expr: recentCancer.doc.length == 5

  - name: topFacetRecent
    expr: recentCancer.facet["year"][0].count > 1000

  - name: medianYearRecent
    expr: recentCancer.numStat["year"].percentiles[1].value > 2021
```

```bash
zuliatest --testConfig test_config.yaml --testOutput results.csv
```
