+++
title = "Unexpected indexes on PostgreSQL 12"
date = 2020-05-14
[extra]
draft = true
+++

I recently [wrote on
Twitter](https://twitter.com/murkt/status/1258049654601719808) that we've
upgraded our database from PostgreSQL 11 to 12. For a context, the size of the
database itself is 1.5 TB, biggest table has more than one billion rows and
weighs 300+ GB alone, without indexes. Many other tables have 100M+ rows too.

The main benefit that I was expecting from new version is `REINDEX
CONCURRENTLY`. We have a couple of places where we do it manually: `CREATE
INDEX CONCURRENTLY`, drop old index, rename new one. It's a small price to pay
to have that particular task queue directly in our database, with transactional
integrity, JOINs, history and everything.

When I say we do it manually, of course I mean that we have a task that's run
daily. I'm not a total maniac.

We also have a couple of materialized views that we update concurrently in the
same way: `CREATE TABLE` from a query, drop old table, rename new one,
continue. PostgreSQL 13 probably will [get
something](https://commitfest.postgresql.org/27/2138/) on that front too, but
for now we have to set the sun below the horizon manually. In PostgreSQL 11 it
took 22-25 minutes to refresh each of them (this one runs weekly). In PostgreSQL
12 we've got JIT compiler, so I expected some speedups here. From my testing
it's got faster approximately by 10% (2 minutes).

The other thing I remembered was that B-tree indexes should have gotten
better. I found [this
article](https://www.cybertec-postgresql.com/en/b-tree-index-improvements-in-postgresql-v12/)
to be a pretty good explanation. Shortly, new format is introduced that's more
compact, less bloaty, hence faster. And we need to `REINDEX` old indexes after
`pg_upgrade` to have these goodies. Ok, I thought.

After the upgrade, one of my developers suggested to try to convert one hash
index to B-tree on our biggest table. We've had quite a few since PG10 made them
reliable, and when we tested them, hash indexes definitely were smaller. I
didn't think it would help much, but of course we can try and see what
happens. The results were astounding: the index shrank from 38 GB to 25
GB. That's right - "smaller" hash index was taking much more space than "bigger"
btree. This hash index was also freshly rebuilt to check that the difference
stems from innate difference, and not from the bloat of an old index. Before
`REINDEX` this hash index was checking in at hefty 54 GB.

Granted, this column has low cardinality, for 1.2 billion rows it has only 30
millions distinct values (40x difference). Maybe its inefficient with lots of
duplicating values, when they end up in the same hash bucket? I've tested it on
some other table on a column with much higher cardinality, only 2.3 rows per
unique value. Still B-tree was much smaller, only 63% of hash index (4823 MB vs
7655 MB).

Okay, what if I create a hash index on some primary key column and then compare it
with a btree? Still 25% difference in the favor of B-tree. What the heck? Was
our testing back in the day sloppy and we compared apples to oranges, or is this
new B-tree just so much better?

Do we really need hash indexes at all when B-tree indexes are more compact,
faster and have more features? Ok, all previous testing was done on `bigint`
columns, maybe `text` columns have something else to say?

They definitely do: for example, it's impossible to have a btree index on a text
column with value longer than third of a page. If you try to do it, PostgreSQL
will helpfully complain:

```
ERROR: index row size 2744 exceeds btree version 4 maximum 2704
HINT:  Values larger than 1/3 of a buffer page cannot be indexed.
Consider a function index of an MD5 hash of the value, or use full text indexing.
```

Looks like hash indexes do not have such a limit at all. I've got curious and
tried to create a hash index on a text column with a value 15 millions
characters long, and it did it with no problems at all. I guess hash index in
PostgreSQL is true to its name and doesn't store an indexed value within itself.

What else? For some random table and random column I've tried it on, hash index
is almost twice smaller than btree (44%), much faster to construct (33%). It's
also much faster to query: for the same 1000 random values hash index was three
times faster (5 ms vs. 15ms) with everything in cache, and hit only 2577 shared
buffers with btree hitting 6100.

Maybe hash index will also be more efficient with some other field types like
UUID, but they can't be UNIQUE, so it additionally limits their usefulness.

The end of the story: we converted almost all our hash indexes to btree.
