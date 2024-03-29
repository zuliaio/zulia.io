---
layout: post
title: Query Syntax
description: Explaining the Zulia query syntax based on Lucene.
---
# Query Syntax

Zulia's query parser is based on the [lucene flexible query parser](https://lucene.apache.org/core/9_1_0/queryparser/org/apache/lucene/queryparser/flexible/standard/StandardQueryParser.html).  The documentation here is slightly modified with some zulia specific tweaks and improvements.

### Supported query syntax
A query consists of clauses, field specifications, grouping and Boolean operators and interval functions. We will discuss them in order.

## Basic clauses
A query must contain one or more clauses. A clause can be a literal term, a phrase, a wildcard expression or other expression that

The following are some examples of simple one-clause queries:

* `test`
  * selects documents containing the word test (term clause).
* `"test equipment"`
  * phrase search; selects documents containing the phrase test equipment (phrase clause).
* `"test failure"~4`
  * proximity search; selects documents containing the words test and failure within 4 words (positions) from each other. The provided "proximity" is technically translated into "edit distance" (maximum number of atomic word-moving operations required to transform the document's phrase into the query phrase).
* `tes*`
  * prefix wildcard matching; selects documents containing words starting with tes, such as: test, testing or testable.
* `/.est(s|ing)/`
  * documents containing words matching the provided regular expression, such as resting or nests.
* `nest~2`
  * fuzzy term matching; documents containing words within 2-edits distance (2 additions, removals or replacements of a letter) from nest, such as test, net or rests.

## Field specifications
Most clauses can be prefixed by a field name and a colon: the clause will then apply to that field only. If the field specification is omitted, the query parser will use the query fields given with the query.  If fields are not given with the query, it will use the default search fields from the index config.

Note: in Zulia the fields from the original document can be indexed as different field name.  The field name here refers to the indexed field name.

The following are some examples of field-prefixed clauses:

* `title:test`
  * documents containing test in the title field.

* `title:(die OR hard)`
  * documents containing die or hard in the title field.

* `title,abstract:(die AND hard)`
  * documents containing die AND hard in the title or abstract fields.

* `*Title:(die AND hard)`
  * documents containing die AND hard in an indexed field matching *Title, for example: shortTitle, longTitle

## Boolean operators and grouping

The default operator can be set on a query and defaults to OR.  Any terms or clauses that are not joined with an explicit boolean operator are joined with the default.

NOTE: The boolean operators in Lucene do not use the normal precedence rules without so parentheses so please use parentheses when using multiple boolean operators.

* `Spain Germany`
  * with Default Operator OR --> `Spain OR Germany`
  * with Default Operator AND --> `Spain AND Germany`
* `title:(news entertainment)`
  * with Default Operator OR --> `title:(news OR entertainment)`
  * with Default Operator AND --> `title:(news AND entertainment)`

You can combine clauses using Boolean AND, OR and NOT operators to form more complex expressions, for example:

* `test AND results`
  * selects documents containing both the word test and the word results.

* `test OR suite OR results`
  * selects documents with at least one of test, suite or results.

* `title:test AND NOT title:complete`
  * selects documents containing test and not containing complete in the title field.

* `title:test AND -title:complete`
  * same as above with alternate syntax
  * *-* indicates NOT

* `title:(-apple orange)`
  * selects document with title without the word apple but with the word orange

* `title:test AND (pass* OR fail*)`
  * grouping; use parentheses to specify the precedence of terms in a Boolean clause. Query will match documents containing test in the title field and a word starting with pass or fail in the default search fields.

* `title:(pass fail skip)`
  * if the default operator for the query is OR (default), documents containing any of *pass*, *fail* or *skip* in the title field.
  * if the default operator for the query is AND, documents containing all of *pass*, *fail* or *skip* in the title field.

* `title:(+test +result unknown)`
  * *+* indicates or phrase is required
  * if the default operator for the query is OR (default), documents containing both *test* and *result* in the title field.  The *unknown* term will be used for scoring
  * if the default operator for the query is AND, the + for required does not change the query and gives documents containing both *test*, *result*, and *unknown* in the title field.





Note the Boolean operators must be written in all caps, otherwise they are parsed as regular terms.

## Range operators

To search for ranges of textual or numeric values, use square or curly brackets, for example:

* `name:[Jones TO Smith]`
  * inclusive range; selects documents whose name field has any value between Jones and Smith, including boundaries.

* `score:{2.5 TO 7.3}`
  * exclusive range; selects documents whose score field is between 2.5 and 7.3, excluding boundaries.

* `score:{2.5 TO *]`
  * one-sided range; selects documents whose score field is larger than 2.5.

* `pubDate:[2022-01-01 TO 2022-02-01]`
  * Date range example

* `pubDate:[2021-06-21T05:00:00.000Z TO 2021-12-12T08:10:00.000Z]`
  * Date range example with timestamp


## Dates
Dates can be represented the following formats:
* `yyyy`
  * ex. 2021
* `yyyyy-MM` or `yyyy/MM`
  * ex. 2020-05 or 2020/05
* `yyyy-MM-dd` or `yyyy/MM/dd`
  * ex. 2004-04-01 or 2004/04/01
* `yyyy-MM-ddTHH:mm:ss.SSSZ` or `yyyy/MM/ddTHH:mm:ss.SSSZ`
  * ex. 2012-08-03T11:59:00.000Z or 2012/08/03T11:59:00.000Z

Range query dates with / do not special handling, but otherwise they need to be quoted or escaped
* ex:
  * `date:"2012/02/01"`
  * `date:[2012/02/02 TO 2023/03/04]`

Sample queries
* `date:2022-04`
  * equivalent to date:[2022-04-01T00:00:00.000Z TO 2022-04-30T23:59:59.999Z]
* `date:2022-04-02`
  * equivalent to date:[2022-04-02T00:00:00.000Z TO 2022-04-02T23:59:59.999Z]
* `date:[2022-04 TO 2022-05]`
  * equivalent to date:[2022-04-01T00:00:00.000Z TO 2022-05-31T23:59:59.999Z]
* `date:[2022-04 TO 2022-05}`
  * equivalent to date:[2022-04-01T00:00:00.000Z TO 2022-05-01T00:00:00.000Z}
* `date:{2022-04 TO 2022-05}`
  * equivalent to date:{2022-04-01T23:59:59.999Z TO 2022-05-01T00:00:00.000Z}

## Length Operations

Zulia indexes string fields with character length information of the input string and list fields with list length information

* `|title|:[10 TO 20]`
  * searches for titles with character length between 10 and 20 inclusive

* `|||authors|||:4`
  * finds documents where authors list contains four items, i.e. *authors: ["Tom","Sally","Bob", "Jen]*

*

## Term boosting
Terms, quoted terms, term range expressions and grouped clauses can have a floating-point weight boost applied to them to increase their score relative to other clauses. For example:

* `jones^2 OR smith^0.5`
  * prioritize documents with jones term over matches on the smith term.

* `field:(a OR b NOT c)^2.5 OR field:d`
  * apply the boost to a sub-query.


# Special character escaping
Most search terms can be put in double quotes making special-character escaping not necessary. If the search term contains the quote character (or cannot be quoted for some reason), any character can be quoted with a backslash. For example:

```
\:\(quoted\+term\)\:
```
a single search term (quoted+term): with escape sequences. An alternative quoted form would be simpler:
```
":(quoted+term):"
```

## Minimum-should-match constraint for Boolean disjunction groups
A minimum-should-match operator can be applied to a disjunction Boolean query (a query with only "OR"-subclauses) and forces the query to match documents with at least the provided number of these subclauses. For example:

A minimum should match can be set on the query which is equivalent to wrapping the entire query in a minimum should match.

* `(blue crab fish)@2`
  * matches all documents with at least two terms from the set [blue, crab, fish] (in any order).
  *  also can be written as `(blue crab fish)~2`

* `((yellow OR blue) crab fish)@2`
  * sub-clauses of a Boolean query can themselves be complex queries; here the min-should-match selects documents that match at least two of the provided three sub-clauses.
  * also can be written as `((yellow OR blue) crab fish)~2`



# Interval function clauses
Interval functions are a powerful tool to express search needs in terms of one or more * contiguous fragments of text and their relationship to one another. All interval clauses start with the fn: prefix (possibly prefixed by a field specification). For example:

* `fn:ordered(quick brown fox)`
  * matches all documents (in the default field or in multi-field expansion) with at least one ordered sequence of quick, brown and fox terms.

* `title,abstract:fn:maxwidth(5 fn:atLeast(2 quick brown fox))`
  * matches all documents in the title or abstract field where at least two of the three terms (quick, brown and fox) occur within five positions of each other.

* Fields indexed with the name `fn` (not recommended) can be searched by setting fn as one of the default search fields but not by simply prefixing the query with the field name.  To use prefix a term or term grouping, use the multi field syntax with a blank additional field like `fn,:someTerm`.

Please refer to the [interval functions package](https://lucene.apache.org/core/9_1_0/queryparser/org/apache/lucene/queryparser/flexible/standard/nodes/intervalfn/package-summary.html) for more information on which functions are available and how they work.


# Zulia specific query syntax


## Numeric Set
Numeric set queries are more efficient for sets of numbers than a standard filter or scored query. They work on integer, long, float, and double fields.
They can be done as a separate query since 2.7.0 (NumericSetQuery in Java) and inline with a query using the syntax below since 3.4.1.
* `intField:zl:ns(10 4 25)`
  * searches intField for documents containing any of the numbers 4,10,25
* `intField1,intField2:zl:ns(10 4 25)`
  * searches both intField1 and intField2 for documents containing any of the numbers 4,10,25
* `zl:ns(1 2 3 4)`
  * searches the default search fields for documents containing any of the numbers 4,10,25 (all default search fields must be numeric)


## Terms In Set Query
Term in set queries are a more efficient way to search a set of terms as they appear in the search index (tokenized and filtered for non keyword analyzers). If a keyword analyzer is used terms will be case-sensitive and need to be a complete match.  
They can be done as a separate query since 1.7.35 (TermQuery in Java) and inline with a query using the syntax below since 3.4.1.

* `id:zl:tq(abc 123 xyz rrr)`
  * searches field id for analyzed terms abc, xyz, or rrr.  If id field was analyzed as lower case keyword this would match ids ABC or Xyz.
* `mainTopic,subTopic:zl:tq("Lung Cancer" "Brain Cancer" Diabetes)`
  * searches field mainTopic and subTopic for terms "Lung Cancer" "Brain Cancer" Diabetes, returning documents if either field matched any term.  In this case topics and subTopics are assumed to be keyword fields.

## Notes
* Fields indexed with the name `zl` (not recommended) can be searched by setting zl as one of the default search fields but not by simply prefixing the query with the field name.  To use prefix a term or term grouping, use the multi field syntax with a blank additional field like `zl,:someTerm`.

