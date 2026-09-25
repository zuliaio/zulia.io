---
layout: post
title: Signals Library
description: Record what users do and answer usage questions from a Zulia index
---

# Zulia Signals Library

The `zulia-signals` module records what the users of an application do (searches, clicks, views, exports, edits) as signals stored in a Zulia index, and answers usage questions from that index with facet based reports. Search signals also double as a query log: what people searched for, what they were shown, and what they clicked.

Signals are recorded through the regular Java client, so the only infrastructure is a Zulia index. No separate analytics store is involved.

**Beta.** `zulia-signals` ships in Zulia 5.4.0 as a beta module. The API and the stored schema may change in a later 5.x release based on feedback from the first deployments. Changes will be called out in the release notes.

This page describes the module as of 5.5.2. Coming from 5.5.1: `ActorActivity` is now `ActorTally<Activity>`, `SignalsIndexConfig.dimensions` and `dimension` are `indexTags` and `indexTag` (the old names still work, deprecated), and `record` also takes a `Signal.Builder`. The stored schema is unchanged.

## Gradle
```bash
repositories {
    mavenCentral()
}

dependencies {
    implementation 'io.zulia:zulia-signals:5.5.2'
}
```

## Maven
```xml
<dependency>
    <groupId>io.zulia</groupId>
    <artifactId>zulia-signals</artifactId>
    <version>5.5.2</version>
</dependency>
```

# Signals

A `Signal` is one thing an actor did in an application. It carries:

* `app` (required): the application recording the signal. Reports are scoped by app.
* `actor` (required): who did it, an `Actor` with an id and an `ActorType`.
* `action` (required): what they did, an open string. `Actions` holds the common ones.
* `target`: what they did it to, a type and one or more ids. `Targets` holds the common types.
* `client` and `session`: the channel (web, mobile, api) and a session id, both optional. When the only session handle is a secret such as the bearer token, `sessionHashed(token)` stores a SHA-256 derived id instead of the token.
* `search`: query text, indexes, result count, latency, impressions, and click position for search and click signals.
* `duration`: an elapsed time for exit style signals such as logout.
* `tags`: app specific key and value pairs. Every tag is stored. An indexed tag key can be filtered, counted, and tallied on.

`signalId` (a random UUID) and `timestamp` default at build time and can be set explicitly, for example when importing old logs.

## Actors

| Factory | Type | Meaning |
|:--|:--|:--|
| `Actor.user(id)` | USER | A signed in person |
| `Actor.anonymous(sessionId)` | ANONYMOUS | A person who is not signed in, identified by their session |
| `Actor.agent(id, delegatedBy)` | AGENT | An AI agent acting for the person in `delegatedBy` |
| `Actor.service(id)` | SERVICE | Another application calling under its own identity, such as an API key owner |
| `Actor.system()` | SYSTEM | The platform itself: jobs, reindexes, health checks |

Reports count USER by default. SERVICE is usage and counts when asked for. SYSTEM is platform load and is excluded from every report unless `includingSystem()` is called.

## Actions and Targets

`Actions` and `Targets` are constant interfaces, not enums, so an application adds its own action and target types without a library change.

```java
// session: login, logout, heartbeat
// navigation: visit (a page or workspace), view (one record)
// content: create, update, delete, upload
// curation: annotate, assign
// search: search, click, export, save
Actions.SEARCH

// index, project, record, document, page, user, field, dataset, preferences
Targets.DOCUMENT
```

# Recording Signals

## Setup

`SignalsIndexConfig` describes the index and which tags it indexes. `SignalsClient` records signals through a `ZuliaWorkPool`.

```java
ZuliaWorkPool pool = new ZuliaWorkPool(ZuliaPoolConfig.localhost());

SignalsIndexConfig config = SignalsIndexConfig.defaults()
        .indexName("signals")                        // the default
        .zone(ZoneId.of("America/New_York"))         // zone for day, week, and month buckets, UTC by default
        .indexTags("category", "plan")               // tag keys to filter, count, and tally on
        .indexTag("items", SignalField.Kind.LONG);   // a numeric tag key for range filters

SignalsClient signals = new SignalsClient(pool, config)
        .onFailure(RecordFailurePolicy.LOG_AND_DROP)
        .stamping(builder -> builder.tag("version", "2.4.1"));

// optional, fails fast at startup if Zulia is unreachable. record creates the index on first use otherwise.
signals.ensureStorage();
```

The client creates the index the first time it is needed. Calling `ensureStorage()` at startup is only for failing fast. The index is created with one shard so every facet count is exact.

## Building and Recording

```java
Signal viewed = Signal.builder()
        .app("shop").client("web").session(sessionId)
        .actor(Actor.user("u-1042"))
        .action(Actions.VIEW)
        .target(Targets.DOCUMENT, "sku-42")
        .tag("category", "shoes")
        .build();

RecordResult result = signals.record(viewed);
result.signalId();   // the stored id
result.accepted();   // false when the signal was dropped under LOG_AND_DROP

signals.recordAll(List.of(viewed, another));
```

`build()` names any missing required value in its error. `record` also accepts the builder itself and builds under the failure policy, see below. Recording never mutates the signal, so the same `Signal` can be logged, recorded, and passed on.

### Tags

Tags carry the facts an application cares about. Values can be a string, long, boolean, enum (stored by name), `Instant`, or a list of strings. An indexed list tag facets once per element.

```java
Signal.builder()
        .app("shop").actor(Actor.user("u-1042")).action(Actions.EXPORT)
        .target(Targets.DATASET, "orders")
        .tag("format", ExportFormat.CSV)       // enum name
        .tag("items", 1200L)                   // numeric, declared LONG above
        .tag("compressed", true)
        .tag("columns", List.of("id", "total", "placed"))
        .tagIfPresent("plan", user.plan())     // skips null or blank
        .duration(Duration.ofSeconds(4))
        .build();
```

Every tag is stored. Only tag keys named in `indexTags` or `indexTag` are indexed, and only those can be used in reports. The index is created with the keys indexed at that time. A key indexed later is pushed to the index the next time the client sets up storage, and reaches the signals recorded before that only by reindexing. A tag key must not contain `.` or `$`. A tag indexed with a kind must carry a matching value: a LONG tag rejects a string, an INT tag rejects a value that does not fit an int.

### Bulk Targets

One action on many records is one signal with a list of target ids. Reports by target count each id once and reports by action count the signal once.

```java
Signal.builder()
        .app("shop").actor(Actor.user("u-1042")).action(Actions.UPDATE)
        .target(Targets.DOCUMENT, List.of("sku-42", "sku-43", "sku-44"))
        .tag("field", "price")
        .build();
```

Target ids are deduplicated in order and capped at 10,000 per signal. Past that, record the count as a tag or split the bulk.

## Failure Policy

The failure policy decides what happens when Zulia is unreachable while recording. It covers `record`, `recordAll`, and `ensureStorage`. Every other method throws.

| Policy | Behavior |
|:--|:--|
| `PROPAGATE` (default) | `record` stores the signal synchronously and throws the unchecked `SignalsException` on failure. For setup and batch code. |
| `LOG_AND_DROP` | `record` hands the signal to one background writer thread and returns at once. Failures are logged, the first with a stack trace and then one summary line per minute, and counted in `droppedSignals()`. For request paths, so an outage never breaks the search it is measuring. |

Under `LOG_AND_DROP` the writer queue holds 10,000 signals. Past that, `record` drops the signal, returns a result with `accepted() == false`, and counts the drop. `queuedSignals()` reports the queue depth. `close()` drains the queue for up to five seconds, so call it at shutdown.

Under `LOG_AND_DROP` nothing about recording throws. `record(Signal.Builder)` builds inside the policy, so a signal that fails validation, a stamp that breaks it, or a tag the index schema rejects is a counted drop with `accepted() == false`, and the request path needs no try and catch of its own. A signal that never built has a null id on its result. Under `PROPAGATE` those validation errors are thrown as they are, since they are programming errors.

```java
signals.record(Signal.builder().app("shop").actor(actor).action(Actions.VIEW).target(Targets.PAGE, "home"));   // never throws under LOG_AND_DROP
```

```java
SignalsClient signals = new SignalsClient(pool, config).onFailure(RecordFailurePolicy.LOG_AND_DROP);
// ...
long dropped = signals.droppedSignals();   // a gauge for monitoring
signals.close();
```

## Session Ids from Secrets

A session id groups one login's signals. Applications on bearer tokens often have no other session handle, and the token must never land in the index, since anyone who can read the index could replay it. `SessionIds.hashed` derives a stable 32 character id from any secret, and the builder's `sessionHashed` applies it in place.

```java
Signal.builder().app("shop").actor(Actor.user("u-1042")).sessionHashed(bearerToken)   // stores 5f4dcc3b..., never the token
String sessionId = SessionIds.hashed(bearerToken);                                   // the same id, for passing around
```

A refreshed token starts a new session. A null or blank secret leaves the session unset. A plain hash is enough for a high entropy secret like a signed token. Use the actor id mapper's keyed HMAC for guessable values such as an email.

## Pseudonymizing Actor Ids

An `ActorIdMapper` maps actor ids before storage. The identity mapper is the default. `hmacSha256` stores a keyed hash instead of the id, applied to both `actorId` and `delegatedBy`. The hash includes the app name, so the same person gets a different pseudonym in each app and two apps' signals cannot be joined on the actor.

```java
byte[] key = ...;   // at least 16 bytes, kept out of the index and the source tree
SignalsClient signals = new SignalsClient(pool, config, ActorIdMapper.hmacSha256(key));
```

Active user counts and per actor reports work the same way on the pseudonyms.

# Search Signals

`ZuliaSearchSignals` turns a Zulia `Search` and its `SearchResult` into a pre-filled search signal. The application adds the app, client, session, and actor.

```java
Search search = new Search("products").setAmount(10);
search.addQuery(new ScoredQuery("running shoes").addQueryFields("title", "description"));
search.addQuery(new FilterQuery("inStock:true"));
search.addCountFacet(new CountFacet("brand"));
SearchResult result = pool.search(search);

Signal searched = ZuliaSearchSignals.searchSignal(search, result)
        .app("shop").client("web").session(sessionId).actor(Actor.user("u-1042"))
        .build();
signals.record(searched);
```

The search signal carries:

* The query text of the scored clauses. Filter, term list, and vector clauses are counted, not stored as text, so filters never read as something the user typed.
* The indexes searched, as a multi valued facet and as the signal's targets, so a multi index search counts once per index.
* The result count and the latency, which defaults to the result's own client round trip time. Pass a `Duration` to `searchSignal` to record a span the application measured itself.
* The first 20 result ids as impressions, stored but not indexed, so click through has denominators.
* The paging offset and size, the sort fields, the query fields, the number of facets requested, and the search label, all as tags.

When the user acts on a result, derive the follow-up from the search signal. It carries the actor, session, and client forward and links back through `searchSignalId`. It carries no query text, so top query reports count searches only.

```java
// the user opened the fourth result
signals.record(ZuliaSearchSignals.click(searched, "sku-42", 3).build());

// any other action on a result, with the target type of the thing itself
signals.record(ZuliaSearchSignals.linked(searched, Actions.SAVE, Targets.DOCUMENT, "sku-42", 3).build());
```

## Saved Searches

`searchSignalId` names one execution. When the application has its own key for a search definition, a saved search or a search the client passes back in the URL, set it as `searchId`. Every execution, page, and follow-up then joins on the key with no server side state.

```java
Signal searched = ZuliaSearchSignals.searchSignal(search, result)
        .app("shop").actor(Actor.user("u-1042"))
        .searchId("saved-7")
        .build();

Signal paged = Signal.builder().app("shop").actor(Actor.user("u-1042")).action("page").searchId("saved-7").tag("offset", 20L).build();
```

# Storage

## Index Schema

Every signal is stored as one document with these fields. Built in fields live at the top level and tags live in a `tags` sub document. Custom searches name an indexed tag as `tags.<key>`.

| Field | Type | Indexed as | Used for |
|:--|:--|:--|:--|
| signalId | STRING | keyword, the unique id | idempotent ingest |
| timestamp, receivedAt | DATE | sortable | ordering, range filters, retention |
| day, week, month | STRING | keyword facet | time series without a date histogram, stamped in the configured zone |
| app, client, actorType, actionType, targetType | STRING | keyword facet | every report's group by, `by` and `tallyBy` |
| actorId, targetId | STRING, targetId a list for a bulk | keyword facet | active users, distinct targets |
| sessionId, delegatedBy | STRING | keyword | sessionization, drill down |
| searchQuery | STRING | standard analyzer | the query log, searchable |
| searchQueryNormalized | STRING | keyword facet | top and zero result queries, lower cased with whitespace collapsed |
| searchIndex | list of STRING | keyword facet | per index search counts |
| searchResultCount, searchLatencyMs, searchClickedPosition | NUMERIC | sortable | zero result rate, latency, click position |
| searchClickedDocId, searchSignalId, searchId | STRING | keyword | click linkage, saved search joins |
| searchShownDocIds | list of STRING | stored only, at most 20 | impressions |
| durationMs | NUMERIC | sortable | session length, dwell time |
| tags.&lt;key&gt; | indexed kind | keyword facet by default | indexed tags, the app's own report fields |
| tags | sub document | stored | every tag, indexed or not |

The `SignalField` enum carries each built in field's name and kind, so reports and custom searches can name fields without string literals.

## Monthly Partitions

By default every signal lands in one index and retention deletes by timestamp range. For long retention or high volume, opt in to one index per month behind an alias.

```java
SignalsIndexConfig config = SignalsIndexConfig.defaults().indexName("signals").monthlyPartitions();
```

The client then creates `signals-2026-09` and so on, with the alias `signals` spanning every month and its write index on the current month. The client moves the write index forward on the first signal of a new month, and reports read through the alias so they see every month. Retention drops whole months instead of paging deletes.

The alias is the partition registry. `existingPartitions()` lists its members, `ensurePartition(YearMonth)` adds a past month before a backfill, and `dropPartition(name)` removes a month from the alias and deletes it, refusing to drop the current write index.

Partitioning cannot be switched on for an existing single index of the same name, since the alias and the index would collide. Use a new index name or migrate.

# Reports

## Usage Reports

`UsageReports` answers the usual questions with facet requests. Every report is scoped to one app and one `TimeRange`, from inclusive and to exclusive. Build ranges from dates in the index zone rather than by hand, since an end at the start of today silently drops today.

```java
UsageReport sinceLaunch = reports.of("shop", LocalDate.of(2026, 1, 1));      // start of that day in the index zone, up to now
UsageReport september = reports.of("shop", YearMonth.of(2026, 9));           // one calendar month in the index zone
TimeRange window = TimeRange.days(LocalDate.of(2026, 9, 1), LocalDate.of(2026, 9, 15), zone);   // whole days, last day included
TimeRange recent = TimeRange.lastDays(30);                                   // rolling, clock based
```

```java
UsageReports reports = new UsageReports(signals);
UsageReport shop = reports.of("shop", TimeRange.lastDays(30));

long activeUsers = shop.activeUsers();                                        // distinct USER actors
long everyone = shop.activeUsers(ActorType.USER, ActorType.ANONYMOUS);        // signed in and not
List<DimensionCount> byCategory = shop.by("category");                        // an indexed tag
List<DimensionCount> byClient = shop.by(SignalField.CLIENT);                  // a built in field
List<DimensionCount> daily = shop.overTime(Bucket.DAY);                       // chronological, DAY, WEEK, or MONTH
List<DimensionCount> topQueries = shop.topQueries();                          // normalized query text by frequency
List<DimensionCount> zeroResult = shop.zeroResultQueries();                   // queries that found nothing
long productsViewed = shop.distinct(Targets.DOCUMENT, Actions.VIEW);          // distinct target ids for one action
long productsTouched = shop.distinct(Targets.DOCUMENT);                       // for any action
List<DimensionCount> perApp = reports.byApp(TimeRange.lastDays(30));          // the cross app rollup
```

Each grouped report returns `DimensionCount(value, count)` rows by count descending. `by(String)` accepts an indexed keyword tag and `by(SignalField)` a built in keyword facet field. A tag that is not indexed, or is indexed as a number, throws and names the indexed keys.

SYSTEM actors are excluded from every report. To see platform load next to usage:

```java
List<DimensionCount> load = shop.includingSystem().by(SignalField.ACTOR_TYPE);
```

Counts are exact up to 50,000 values per field. A grouped report past that limit returns the top values. A distinct count or a tally past it throws instead of returning a truncated number. `maxFacetValues` raises or lowers the limit. Every value comes back in one response, so a higher limit costs response size on every report.

```java
UsageReports reports = new UsageReports(signals).maxFacetValues(200_000);
```

## Per Actor Tallies

`forActivity` narrows a report to one action, optionally on one target type. `forActor` narrows it to one actor. `forTag` narrows it to one value of an indexed tag, and `forField` to one value of a built in keyword field such as the client. The narrowings stack, each at most once per field. `tally` counts signals per actor for a list of activities, which is the usual activity table with one column per activity and one row per actor that did at least one of them.

```java
Activity projectsCreated = Activity.of(Actions.CREATE, Targets.PROJECT);
Activity recordsVisited = Activity.of(Actions.VISIT, Targets.RECORD);
Activity logins = Activity.of(Actions.LOGIN);                                  // any target or none

List<ActorTally<Activity>> rows = shop.tally(projectsCreated, recordsVisited, logins);  // by total descending
for (ActorTally<Activity> row : rows) {
    System.out.println(row.actorId() + " " + row.count(projectsCreated) + " " + row.count(recordsVisited) + " " + row.count(logins));
}

List<DimensionCount> creators = shop.forActivity(projectsCreated).by(SignalField.ACTOR_ID);   // one column
List<ActorTally<Activity>> oneRow = shop.forActor("u-1042").tally(projectsCreated, recordsVisited);      // one row, empty when the actor did none
long recordsSeen = shop.forActor("u-1042").distinct(Targets.RECORD, Actions.VISIT);            // distinct records for one actor
List<ActorTally<Activity>> inShoes = shop.forTag("category", "shoes").tally(projectsCreated, logins);   // the table inside one tag value
```

A tally runs one search per activity, so its cost does not grow with the number of actors. It counts signals, so a bulk signal counts once. Distinct targets per actor go through `forActor` and `distinct`. `forActor` takes the real actor id and maps it the way the client stored it, so it works under a pseudonymizing `ActorIdMapper`. The `actorId` on each row is the stored id, so under `hmacSha256` it is the pseudonym and cannot be turned back into the real id. Keep the activities disjoint. An activity without a target type overlaps every activity of that action, and the total that orders the rows is the sum of the columns. A report narrows to one activity and one actor at most once, so `tally` runs on a report that is not already narrowed by `forActivity`. A tally with at least `maxFacetValues` actors for one activity throws rather than drop actors.

## Per Actor Tallies by Tag

`tallyBy` counts signals per actor and per value of an indexed keyword tag, or of a built in keyword facet field such as the client or a time bucket. Narrow by `forActivity` first to break one activity down. Each row is an `ActorTally<String>(actorId, counts)` with one entry per value the actor has a signal for, by total descending. `tally` returns the same record as `ActorTally<Activity>`, so both tables share `count(column)` and `total()`.

```java
UsageReport adds = shop.forActivity(Activity.of(Actions.CREATE, Targets.DATASET));

List<ActorTally<String>> rows = adds.tallyBy("category");                    // who created what, per category
for (ActorTally<String> row : rows) {
    for (var entry : row.counts().entrySet()) {
        System.out.println(row.actorId() + " added " + entry.getValue() + " " + entry.getKey());
    }
}

List<ActorTally<String>> perDay = adds.tallyBy(Bucket.DAY.field());          // per actor per day
List<ActorTally<String>> perClient = shop.tallyBy(SignalField.CLIENT);       // every signal, per actor per client
```

A tally by tag runs one search per actor or one per value, whichever there are fewer of, plus two to count them, so its cost is `2 + min(actors, values)`. It throws past `maxTallySearches` searches, 1,000 by default, naming both cardinalities, and past `maxFacetValues` actors or values. Narrow the report, raise the cap on `UsageReports`, or run a custom search for a wider table. A report narrowed by `forActor` or by `forTag` on the same tag refuses `tallyBy`, since one row or one column is a `by`. A list tag counts a signal once per element, so a row's total can exceed the actor's signals.

## Custom Queries

Anything the reports do not cover is a regular Zulia search against the signals index. The client runs it against the right name in both storage modes.

```java
SearchResult exportsByCategory = signals.search(search -> {
    search.setAmount(0);
    search.addQuery(new FilterQuery("actionType:export"));
    search.addQuery(new InstantRangeFilter("timestamp").setMinValue(Instant.now().minus(Duration.ofDays(7))));
    search.addCountFacet(new CountFacet("tags.category"));
});
```

# Retention

`SignalsRetention` removes signals older than a cutoff. In single index mode it pages through the timestamp range and batch deletes. In partitioned mode it drops every month before the month of the cutoff. The result reports both.

```java
RetentionResult result = new SignalsRetention(signals).deleteBefore(Instant.now().minus(Duration.ofDays(400)));
result.signalsDeleted();      // in both modes
result.partitionsDropped();   // the month names, empty in single index mode
```

Retention always throws on failure, whatever the failure policy, since it is an operator path.
