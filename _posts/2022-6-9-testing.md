---
layout: post
title: Testing
description: Validate search results with the Zulia test framework
---

The `zuliatest` command is a YAML-driven testing framework for validating Zulia search results. It executes queries against a Zulia cluster and evaluates JavaScript expressions to assert expected results, producing a CSV report.

This is useful for search quality regression testing, CI/CD integration, and validating index data after migrations.

## Running Tests

```bash
zuliatest --testConfig /path/to/test_config.yaml --testOutput /path/to/results.csv
```

### Options
```
--testConfig      Full path to the test config YAML file (required)
--testOutput      Full path to the test output CSV file (required)
--reportOutput    Full path to write the run report JSON (timestamp, recorded values, test outcomes)
--previousReport  Full path to an earlier run report JSON. Its values are exposed to expressions as saved.name
--recordBaseline  First run of a config that uses saved. Tests and values that use saved are marked SKIPPED. Cannot be combined with --previousReport
--asOf            Date (yyyy-MM-dd) that now and the date aliases resolve from. Defaults to today
--var             name=value, sets or overrides a config variable. Repeatable
--showStack       Show stack traces on errors
```

The options from `--reportOutput` to `--var` are new in 5.7.0.

### Exit Codes
* `0` - Every test passed. Skipped tests do not count against the run
* `1` - Every expression was evaluated and at least one test failed. Also returned when the run stops early, on a connection failure or on a config problem such as a duplicate test name or an unknown `${...}` placeholder in a query string
* `2` - At least one test or value could not be evaluated (ERROR), or the command line arguments were invalid
* `9` - The config file or the `--previousReport` file was not found or not readable

Before 5.7.0 a bad expression stopped the run with exit code 1. It is now reported as ERROR for that test, the run continues, and the exit code is 2.

## YAML Configuration

The test configuration has four main sections: connections, indexes, searches, and tests. The optional sections `variables`, `script`, `functions` and `values` are described below.

A schema is published for editor validation and completion. Put this line at the top of the config to use it:

```yaml
# yaml-language-server: $schema=https://zulia.io/schema/zulia_testing_schema-1.2.json
```

Schema 1.2 covers the sections added in 5.7.0. Schema 1.1 stays published for older configs. Schema 1.2 lists `queries`, `topN`, `tests`, `logSearches` and `logSearchResults` as required, so a schema validating editor may flag their absence even though the runner accepts a config without them.

### Connections

Define one or more Zulia server connections.

```yaml
connections:
  - name: local
    serverAddress: localhost
    port: 32191
  - name: production
    serverAddress: 10.0.0.50
    routingEnabled: true
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| name | yes | | Unique connection identifier |
| serverAddress | yes | | Hostname or IP address |
| port | no | 32191 | Zulia gRPC port |
| routingEnabled | no | false | Discover the cluster nodes and send each request to the node that owns the shard. Leave off when serverAddress is a port forward or a load balancer, otherwise requests go to the cluster's own addresses |

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
| queries | no | | List of query clauses. The runner runs the search without a query clause when omitted |
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
| topN | no | 10 | Number of top facet values to return. 0 or omitted means unset and the server default of 10 applies |

#### Stat Facet Fields

| Field | Required | Description |
|-------|----------|-------------|
| facetField | yes | Field to facet on |
| numericField | yes | Numeric field for statistics |
| topN | no | Number of top facet values. 0 or omitted means unset and the server default of 10 applies |

#### Numeric Stat Fields

| Field | Required | Description |
|-------|----------|-------------|
| numericField | yes | Numeric field name |
| percentiles | no | List of percentile points (0.0 to 1.0) |
| percentilePrecision | no | Precision for percentile calculation |

### Tests

JavaScript expressions that validate search results. Each expression must evaluate to `true` (PASS) or `false` (FAIL). An expression that returns anything else, or that cannot be evaluated, is reported as ERROR and the run continues. Test names must be unique.

```yaml
tests:
  - name: hasResults
    expr: allDocs.count > 0

  - name: mostHaveTitles
    expr: cancerDocs.count > allDocs.count * 0.5

  - name: topYearRecent
    expr: cancerDocs.facet["year"][0].label == "2024"
```

### Date Templates

Query strings (`q`) and test and value expressions may contain `${...}` placeholders that resolve to a date. The anchor is today, or the date given with `--asOf`.

Date math works on whole dates. Units are `d`, `w`, `M` (month) and `y`. Hours and smaller units are not supported. A trailing `/unit` rounds down to the start of that day, week (Monday), month or year. With `--asOf 2026-10-22`:

```
${now}            2026-10-22
${now-1y}         2025-10-22
${now-90d}        2026-07-24
${now-1y/y}       2025-01-01   (start of last year)
${now/M}          2026-10-01   (start of this month)
```

Aliases for the common cases:

| Alias | Date math |
|-------|-----------|
| today | now |
| yesterday | now-1d |
| oneWeekAgo | now-1w |
| oneMonthAgo | now-1M |
| oneYearAgo | now-1y |
| startOfWeek | now/w |
| startOfMonth | now/M |
| startOfYear | now/y |
| startOfLastMonth | now-1M/M |
| startOfLastYear | now-1y/y |

An optional `|pattern` suffix formats the date with a Java DateTimeFormatter pattern. The default is `yyyy-MM-dd`, which Zulia accepts directly in range queries and as a single day.

```yaml
searches:
  - name: pubsThisYear
    index: pubs
    queries:
      - q: "pubDate:[${startOfYear} TO ${today}]"
        queryType: FILTER

  - name: pubsSamePointLastYear
    index: pubs
    queries:
      - q: "pubDate:[${startOfLastYear} TO ${oneYearAgo}]"
        queryType: FILTER

  - name: pubsLastYear
    index: pubs
    queries:
      - q: "year:${oneYearAgo|yyyy}"

tests:
  - name: yearToDateTracksLastYear
    expr: pubsThisYear.count > pubsSamePointLastYear.count * 0.8

  - name: topYearIsLastYear
    expr: cancerDocs.facet["year"][0].label == "${oneYearAgo|yyyy}"
```

An unknown name such as `${tomorrow}` in a query string stops the run before any search, with the list of aliases. In a test or value expression it is an ERROR for that test or value, and the run continues. Month arithmetic clamps to the end of the month, so `now-1M` from March 31 is February 28.

Because `${...}` is a placeholder here, a JavaScript template literal cannot be used in a test or value expression. It is read as an unknown placeholder and reported as ERROR. Put that code in a `script` block or a function `body`, which are not templated.

### Variables and Script

`variables` is a map of globals. Every test, value and function sees them by name, and query strings can use them as `${name}`. Values keep their YAML type. String values may contain date templates and earlier variables, in declaration order.

```yaml
variables:
  tolerance: 0.2
  thisYearRange: "[${startOfYear} TO ${today}]"
  lastYearRange: "[${startOfLastYear} TO ${oneYearAgo}]"

searches:
  - name: pubsThisYear
    index: pubs
    queries:
      - q: "pubDate:${thisYearRange}"
        queryType: FILTER

  - name: pubsSamePointLastYear
    index: pubs
    queries:
      - q: "pubDate:${lastYearRange}"
        queryType: FILTER

tests:
  - name: yearToDateTracksLastYear
    expr: Math.abs(pubsThisYear.count - pubsSamePointLastYear.count) <= pubsSamePointLastYear.count * tolerance
```

Override or add a variable from the command line with `--var tolerance=0.1`. The value is read like a YAML scalar, so numbers, true/false and bare dates keep their type. Quote the value to force text. A map or list is rejected.

`script` is JavaScript evaluated once in the global scope before the functions and tests. Use it for constants, helpers or anything else that several tests share. A script is not run through `${...}` templating, because that is JavaScript template literal syntax. Call `date()` instead, which takes the same date expressions and an optional format.

```yaml
script: |
  const MILLION = 1000000;
  const lastYear = date("oneYearAgo", "yyyy");
  function ratio(a, b) { return b === 0 ? 0 : a / b; }

tests:
  - name: atLeastMillion
    expr: allDocs.count >= MILLION

  - name: topYearIsLastYear
    expr: cancerDocs.facet["year"][0].label == lastYear

  - name: cancerShareBelowHalf
    expr: ratio(cancerDocs.count, allDocs.count) < 0.5
```

A variable name must be a valid JavaScript identifier. It may not reuse a date alias, `now`, `saved`, `date`, a search name, a function name or a JavaScript built-in such as `JSON`.

### Functions

Helpers defined once and callable from every test and value expression. Each has a `name`, an optional `args` list, and exactly one of `expr` (a single expression whose result is returned) or `body` (statements with an explicit `return`).

```yaml
functions:
  - name: within
    args: [actual, expected, tolerance]
    expr: Math.abs(actual - expected) <= expected * tolerance

  - name: topLabel
    args: [search, field]
    body: |
      const values = search.facet[field];
      return values.length > 0 ? values[0].label : null;

tests:
  - name: yearToDateTracksLastYear
    expr: within(pubsThisYear.count, pubsSamePointLastYear.count, 0.2)

  - name: topYear
    expr: topLabel(cancerDocs, "year") == "2024"
```

An `expr` is templated like a test expression, so it can use date templates. A `body` is a JavaScript block like `script`. It is not templated and can use template literals freely, so use `date()` and the variable globals inside it. Both can read `saved`.

A function name must be a valid JavaScript identifier that is not a reserved word. It may not reuse a search name, a variable name, `saved`, `date`, another function or a JavaScript built-in such as `Math`. A syntax error reports the function name. A runtime error inside a function reports the test that called it.

### Values

Named expressions whose result is recorded instead of asserted. They run after the tests in the same context and are written to the run report when `--reportOutput` is given. A later run can read them back as `saved.name`, see [Comparing Against a Previous Run](#comparing-against-a-previous-run).

```yaml
values:
  - name: totalDocs
    expr: allDocs.count

  - name: topYear
    expr: cancerDocs.facet["year"][0]
```

Numbers, strings and booleans keep their type. Documents, facets and arrays are written as plain JSON. A value that evaluates to undefined (usually a misspelled property), a function, or an infinite number is left out of the report and listed under `unrecordedValues` with the reason, and the run exits with code 2. A typo is never recorded as null and compared against later. Value names must be unique.

### Logging

Optional flags for debugging. The runner treats both as false when omitted.

```yaml
logSearches: true       # Log search queries as they execute
logSearchResults: true  # Log full search results as JSON
```

## Test Expression Reference

Each search result is available as a JavaScript object using the search's `name`. Variables, functions and `saved` are available by name in the same scope. The following properties are available on a search result:

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

## Comparing Against a Previous Run

Pass an earlier run report with `--previousReport`. Its values are available to every test and value expression as `saved`.

```yaml
tests:
  - name: notShrunk
    expr: allDocs.count >= saved.totalDocs

  - name: topYearUnchanged
    expr: cancerDocs.facet["year"][0].label == saved.topYear.label

values:
  - name: totalDocs
    expr: allDocs.count

  - name: topYear
    expr: cancerDocs.facet["year"][0]
```

The first run has no earlier report. Pass `--recordBaseline` instead of `--previousReport`. Every test and value that reads `saved`, directly or through a function, is marked SKIPPED. Everything else runs, and the report is written for the next run to load.

```bash
zuliatest --testConfig config.yaml --testOutput results.csv --reportOutput report-$(date +%F).json --recordBaseline
```

File naming is up to the caller. A scheduled job can write `report-2026-10-22.json` today and read `report-2025-10-22.json` next year:

```bash
zuliatest --testConfig config.yaml --testOutput results.csv \
  --reportOutput report-$(date +%F).json \
  --previousReport report-$(date -d '1 year ago' +%F).json
```

A test or value that reads `saved` without `--previousReport` or `--recordBaseline`, or reads a name that is not in the report, is reported as ERROR. The message lists the names that are available, the rest of the run continues, and the exit code is 2. This is deliberate. In JavaScript a missing property is `undefined` and `count > undefined` is quietly false, which would report a failing test for the wrong reason.

A value can itself read `saved`, for example a change since the previous run. The baseline run skips it, so the first report does not contain it. On the second run the value is recorded, and anything that reads it as `saved` is SKIPPED once more. From the third run on it is available.

A `script` block that reads `saved` at the top level stops a baseline run, because the script runs once before any test. Read `saved` inside a function instead. A JavaScript `try`/`catch` around a read of `saved` catches the skip, so the test runs with whatever the `catch` returns.

## CSV Output

The output CSV has one row per test with two columns, the test name and its status. The status is `PASS`, `FAIL`, `ERROR` or `SKIPPED`. There is no header row. The reason for an ERROR or a SKIPPED is in the log and in the run report.

```
hasResults,PASS
mostHaveTitles,PASS
topYearRecent,FAIL
ratioCheck,ERROR
notShrunk,SKIPPED
```

## Run Report JSON

Written when `--reportOutput` is given. One file is the whole record of a run.

```json
{
  "formatVersion": 1,
  "timestamp": "2026-10-22T14:03:11.412Z",
  "asOf": "2026-10-22",
  "values": {
    "totalDocs": 2814403,
    "topYear": {
      "label": "2024",
      "count": 1812233
    }
  },
  "tests": [
    {
      "name": "hasResults",
      "status": "PASS",
      "message": null
    },
    {
      "name": "ratioCheck",
      "status": "ERROR",
      "message": "Test <ratioCheck> expression <cancerDocs.count / allDocs.count> must evaluate to a boolean but returned <0.12>"
    }
  ],
  "unrecordedValues": []
}
```

`status` is PASS, FAIL, ERROR or SKIPPED. `message` gives the reason for ERROR and SKIPPED and is null otherwise. `unrecordedValues` uses the same shape for values that are missing from `values` because their expression failed or was skipped in a baseline run.

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
