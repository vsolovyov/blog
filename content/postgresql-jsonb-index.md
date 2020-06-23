+++
title = "Indexing JSONB columns in PostgreSQL"
date = 2020-06-23
[extra]
draft = true
+++

Since time immemorial PostgreSQL supports JSON fields and can even index
them. By immemorial I mean this functionality was added in versions 9.2 and 9.4
that are unsupported now. 

I actually perfectly remember the world where PostgreSQL had no JSON support,
because 9.2 [was released](https://www.postgresql.org/docs/9.2/release-9-2.html)
in 2012 and before that I worked in a company that used MongoDB (we suffered
greatly[<sup id="back1">\[1\]</sup>](#note1)). It was an ample bit of marketing
for Mongo back then: "you can store any document without tediously defining an
ungodly schema! You gain so much flexibility!"

Little did we know back then that the world does not work that way, and
relational SQL databases are actually way more flexible than document-oriented
DBs, columnar DBs or whatever.[<sup id="back2">\[2\]</sup>](#note2) Because
often we don't know what exactly are we going to do with the data, and with
relational DBs we can lay out the data however seems reasonable, and then add
indexes to support our use cases. 

In document-oriented DBs you need to lay out your data exactly the way you're
going to query it later. Or else you'll need to migrate data inside your
schema-less database to other layout, which is way more cumbersome and
error-prone than adding some indexes and JOINs. 

Don't trust me - there is an exceptional talk on [Advanced Design Patterns for
DynamoDB](https://www.youtube.com/watch?v=HaEPXoXVf2k) by Rick Houlihan,
Principal Technologist at AWS. He explains that and so much more - it's a very
information-dense presentation with interesting ideas. I found it useful even
though I don't plan to use DynamoDB nor MongoDB in the near future.

Anyway, JSON support was added into PostgreSQL a long time ago, because
sometimes it is useful to store some documents in the database. And it can be
indexed in [two different
ways](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING) -
full <abbr title="Generalized Inverted Index">GIN</abbr> and a special
`jsonb_path_ops` that supports indexing the `@>` operator only. It means
"contains" and can be used like this:

```
SELECT * FROM table WHERE jsonb_field @> '{"employer": {"country": "ZA"}}';
```

Let me tell you a story about how I cleverly used this feature and it bit me in
the ass.

Story time
-----

I am a co-founder at [www.prophy.science](https://www.prophy.science/)
which is a product that can understand, search and recommend scientific papers
and experts. In order to do that well, we need a collection of all scientific
papers, and papers are often provided by many different providers with different
ids. There are [PubMed](https://www.ncbi.nlm.nih.gov/pubmed/) (30M+ articles),
[PubMed Central](https://www.ncbi.nlm.nih.gov/pmc/) (6M+ articles),
[Crossref](https://www.crossref.org/) (80-100M+),
[INSPIRE](https://inspirehep.net/), there are preprint servers like
[arXiv](https://arxiv.org/), [biorXiv](https://www.biorxiv.org/),
[medRxiv](https://www.medrxiv.org/) and many others. 

There is a widespread system of <abbr title="Digital object
identifier">DOIs</abbr> that are used to persistently identify journal articles,
research reports and data sets. It was introduced in the year 2000, and, as many
of these bibliographic databases predate DOI standard, they have their own
identifiers. Sometimes they even cross-link their IDs between different
services, and sometimes they cross-link wrong articles.

Some monitoring services download data from Crossref, Pubmed, PMC and some
others sources, add them and report that they have 180 millions of articles, 220
millions or some other bullshit. We strive to merge the same article from
different sources into one entity with many external identifiers. We called
these identifiers "origin ids" and stored them in a special `jsonb` column, so
one row could have a record like this:

```
{"pubmed": "3782696", "pmc": "24093010", "doi": "10.3389/fnhum.2013.00489"}
```

It was a simple key-value document with a `jsonb_path_ops` index on it. And
whenever we needed to fetch an article by an origin id, we queried it using a
`@>` operator like that:

```
SELECT id FROM articles WHERE origin_ids @> '{"pubmed": "123456"}';
```

It is a bit easier to store ids this way, no need to maintain a separate table
with hundreds of millions of rows.

One problem arose when we tried to query the index with many different origin
ids. There is no `IN` nor `ANY()`, so we stitched lots of `OR`s together:

```
SELECT id FROM articles WHERE 
    origin_ids @> '{"pubmed": "123456"}' OR 
    origin_ids @> '{"pubmed": "654321"}' OR 
    origin_ids @> '{"pubmed": "123321"}' OR 
    origin_ids @> '{"pubmed": "456654"}';
```

Explain everything
-------

And with enough `OR`s the query gets really slow. Why? `EXPLAIN` helpfully says
that it becomes a sequential scan (I shortened output for clarity):

```
EXPLAIN
 SELECT id, origin_ids
   FROM articles
  WHERE origin_ids @> '{"pubmed": "123456"}' OR
        origin_ids @> '{"pubmed": "654321"}' OR
        ....;   - x200
                        QUERY PLAN
------------------------------------------------------------
 Seq Scan on articles  (rows=7805036)
   Filter: ((origin_ids @> '{"pubmed": "123456"}') OR
            (origin_ids @> '{"pubmed": "654321"}') OR   ...x200)
```

Why? For some reason it thinks that this query will return millions of rows. But
one origin id can match at most one article if my data is correct, so 200
filters should only match 0..200 rows. Let's look at `EXPLAIN ANALYZE` to check:

```
EXPLAIN ANALYZE
 SELECT id, origin_ids
   FROM articles
  WHERE origin_ids @> '{"pubmed": "123456"}' OR
        origin_ids @> '{"pubmed": "654321"}' OR
        ....;   - x200
                        QUERY PLAN
------------------------------------------------------------
 Seq Scan on articles  (rows=7805036) (actual rows=200)
   Filter: ((origin_ids @> '{"pubmed": "123456"}') OR
            (origin_ids @> '{"pubmed": "654321"}') OR   ...x200)
```

It does indeed return only 200 rows. Hmmm... Let's check one row:

```
EXPLAIN ANALYZE
 SELECT id, origin_ids
   FROM articles
  WHERE origin_ids @> '{"pubmed": "123456"}';
                        QUERY PLAN
------------------------------------------------------------
 Bitmap Heap Scan on articles  (rows=43038) (actual rows=1)
   Recheck Cond: (origin_ids @> '{"pubmed": "123456"}')
    ->  Bitmap Index Scan on  ... (rows=43038) (actual rows=1)
         Index Cond: (origin_ids @> '{"pubmed": "123456"}')
```

Supposedly 43 thousands rows for only one filter! And 7.8 million rows are 39
thousands times more than 200, which is pretty close. At the time I fired these
queries we had only 43 millions of articles. PostgreSQL [gathers some
statistics](https://www.postgresql.org/docs/current/planner-stats.html) about
values in different columns to be able to produce reasonable query plans, and
looks like it's shooting blanks for this column.

What's the simplest fix? Oftentimes
[`ANALYZE`](https://www.postgresql.org/docs/current/sql-analyze.html) on a table
is enough to fix broken statistics, but this time it didn't help at
all. Sometimes it's useful to adjust how many rows are analyzed to gather
statistics, and it can be adjusted down to per-column basis with [`ALTER TABLE
... ALTER COLUMN ... SET
STATISTICS`](https://www.postgresql.org/docs/current/sql-altertable.html), but
here it had no effect as well.

Since version 10 PostgreSQL supports [`CREATE
STATISTICS`](https://www.postgresql.org/docs/current/sql-createstatistics.html)
to gather complex statistics for inter-column dependencies and whatnot, but our
filter is single-column, no luck here as well.

Contsel
-------

So I dug some more, and more... And found that operator `@>` uses something
called `contsel`. It was mentioned in [PostgreSQL mailing
list](https://www.postgresql.org/message-id/23452.1288288224@sss.pgh.pa.us)
in 2010. I tried to decrypt what `contsel` means and I think it stands for
"contains selectivity". Then I tried searching PostgreSQL sources for [`contsel`
mentions](https://github.com/postgres/postgres/search?q=contsel&unscoped_q=contsel)
and found exactly [one
place](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/geo_selfuncs.c#L80)
in C code which mentions it:

```
Datum
contsel(PG_FUNCTION_ARGS)
{
	PG_RETURN_FLOAT8(0.001);
}
```

0.001? That looks exactly like the ratio between 43 millions rows in the table
and estimated 43 thousands rows in result. However, if we just multiply 43
thousands by 200 filters we should get 8.6 millions, and PostgreSQL estimated
only 7.8M. This discrepancy bothered me for a minute, because I like to
understand things completely, so they won't set me up for unpleasant surprise
later.

After a minute of contemplating the difference I realized that it's probability
in play - PostgreSQL thinks that every filter can match 0.1% of the total number
of rows and they can overlap. The actual math is:

```
1 - 0.999 ** 200 = 1 - 0.819 = 0.181
```

18.1% of 43 millions is 7.8 millions (I'm rounding numbers here). Itch scratched
successfully.

And, depending on the [different
costs](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS)
of various factors in config, Postgres will select either sequential scan or
will use an index. Our first solution was to slice these filters into batches
with no more than 150 of them per query. It worked quite well for a couple of
years.

Domain modelling failure
------

Until we learned that one article could have more than one such external
identifier per type. For example, some pre-print services grant new DOI for each
version. [10.26434/chemrxiv.11938173.v8](https://dx.doi.org/10.26434/chemrxiv.11938173.v8)
has eight of them at the time of writing. And then it has the main DOI without
version
[10.26434/chemrxiv.11938173](https://dx.doi.org/10.26434/chemrxiv.11938173), and
will have another one if it will be published after peer review. There are other
cases for some other identifier types (we call these types "origin name").

We had two options:

- Store origin ids in a separate table with columns `article_id`, `origin_name`
  and `origin_id` with two indexes - one on `article_id` and the other on
  `(origin_name, origin_id)`;
  
- Accommodate many values per key in `jsonb`. Two more possible options here:

  - Many values per key: `{"doi": ["10.26434/3", "10.26434/3.v1"]}`
  - List of pairs: `[["doi", "10.26434/3"], ["doi", "10.26434/3.v1"]]`
  
  Both can be queried with `@>`, but it's getting even more uglier than it was.
  
We actually ended up doing kind of both - we created a separate table that's
much easier to query with many origin ids at once, and we store a list of pairs
in a separate non-indexed column so it's convenient to query.

Separate table speed-up
--------

As a bonus, it's much-much faster to query a btree index with lots of filters
than a GIN one. With a GIN every `@>` turns into a separate Bitmap Index Scan
that costs approximately millisecond for each (0.7-1.2 ms each if in
cache). With a btree index on two columns we construct a query that looks like
this:

```
SELECT article_id FROM articles_origin_ids WHERE
    (origin_name = $1 AND origin_id = ANY($2)) OR
    (origin_name = $3 AND origin_id = ANY($4)) OR
    (origin_name = $5 AND origin_id = ANY($6));
```

Accessing a btree index is faster even by itself, I get 0.07 ms for Bitmap Index
Scan node in `EXPLAIN ANALYZE` for one `origin_name`, `origin_id` pair. And when
we fit many origins into one query, the cost of accessing an index (each AND
turns into a separate Bitmap Index Scan) is getting amortized between tens to
hundreds of ids with the same `origin_name`. It can go as low as 0.01 ms (10 µs)
per `origin_id`. 

That's two orders of magnitude! Thank you very much, I'll take it. If I would
read something like that in the docs, I would go with a separate table right
from the start.

Additional pitfalls
--------------

Current versions of PostgreSQL add new features that can exacerbate these bad
query plans.

For example, when a query is estimated to return millions of rows, it makes
complete sense to fire up parallel workers, but for simple 150 rows - not so
much. For a test query that I'm running to get concrete numbers for this post,
I'm getting speed up from 143 ms to 120 ms with parallelization taking 6
workers.

JIT compiling bad plans
-------------

The other pitfall is a new one to PostgreSQL 12 (I have a post on [how we
upgraded to it](https://vsevolod.net/postgresql-12-btree/)) and it's called
[query JIT compilation](https://www.postgresql.org/docs/current/jit.html).

JIT compiler will think the query is massive enough for JITting and will spend
additional time on it. For my particular test query it spends 150-1500 ms each
time a query is fired (depends on optimization and inlining). It's 1-10 times
slower than the slow query, and it's up to 1000 times slower than a fast query
to a separate table. **1000 times!**

Thankfully, we migrated origin ids to a separate table before PostgreSQL 12 even
came out.

My friends had [the same
problem](https://solovyov.net/blog/2020/postgresql-query-jit/)[<sup
id="back3">\[3\]</sup>](#note3) with JIT compiling a query for more than 13
seconds for a query that usually executes in 30 ms. Upgrade to PostgreSQL 12
brought their site down until they turned JIT off. They also had a `jsonb`
column with an index on it, which inflated estimated rows and cost. However even
without that part the query was big enough to trigger JIT compilation.

I found the same problem in my code just yesterday, when the query was big
enough to trigger JIT compilation, but amount of results wasn't big enough so it
actually slowed things down.

I tried to make PostgreSQL cache JITted code with [prepared
statements](https://www.postgresql.org/docs/current/sql-prepare.html), but I
wasn't successful. It looked to me like JIT was compiling a query each time I
fired `EXPLAIN ANALYZE`, even after more than five times to stabilize a query
plan. I tried to force generic plan for it to cache JITted code, but it still
didn't help.

Possible fix
--------

I don't really know PostgreSQL enough to know how hard it is to add statistics
for `jsonb` fields. Maybe it's possible to somehow extend `CREATE STATISTICS`,
or make `@>` to respect `n_distinct` somehow.

With the JIT triggering very expensive compilation for complex queries with a
small number of rows I think it's best to penalize the cost of enabling JIT on
the basis of how many nodes[<sup id="back4">\[4\]</sup>](#note4) are there in a
query plan. [There are
settings](https://www.postgresql.org/docs/current/jit-decision.html) like
`jit_above_cost`, and with a setting like `jit_node_cost` decision would be made
like this:

```
jit_above_cost > query_plan_cost + jit_node_cost * query_plan_node_count
```

For now I'll just turn JIT off completely in pg_config, and will enable it with
`SET jit = 'on'` only where I know it helps.


#### Notes

[<sup id="note1">\[1\]</sup>](#back1) They still suffer with MongoDB almost a
decade later.

[<sup id="note2">\[2\]</sup>](#back2) Datomic claims that it's even more
flexible than relational DBs, and I played with it a bit and tend to think it
really is. However, their main focus is on Datomic Cloud on AWS, and I need
on-premise, with no additional Cassandra.

[<sup id="note3">\[3\]</sup>](#back3) Author of that post is not only a friend
of mine, but my brother as well.

[<sup id="note4">\[4\]</sup>](#back4) Are they actually called "nodes"? I'm not
sure.
