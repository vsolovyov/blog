+++
title = "PostgreSQL archiver failure"
date = 2021-11-08
[extra]
draft = true
+++

We've got an alert that disk space on a server with production database is
running low.

We have separate partition that stores PostgreSQL data (`/var/lib/postgres/`),
so that nothing can interfere, and **this partition** was running low. Database
size was pretty stable at 3.5 TB, but WALs were accumulating and we already had
2+ terabytes of them. We had `archive_mode = on`, and looks like it wasn't
succeeding for some reason. Our monitoring suggested that it started on
Thursday, around 12:30 UTC. We could afford to lose these WALs, so we tried to
change `archive_command` to `/bin/true` and reload config, but it did nothing
and WALs kept accumulating, instead of disappearing right away.

We turned off write-heavy workload that generated most of those WALs to buy us
more time and tried to understand what was happening. We looked through logs
(nothing), poured through documentation — learned nothing useful from it.

We googled different things and looked through many articles, but they were
talking about other things that we knew, nothing useful for us. Then we stumbled
across [this
one](https://www.percona.com/blog/2020/09/09/why-postgresql-wal-archival-is-slow/
) from Percona blog that has this wonderful image
https://www.percona.com/blog/wp-content/uploads/2020/08/Archiving.png and it
explains how the whole thing works, separate archiver process together with
`***.ready` and `***.done` files.

[![WAL Archiver](/img/pg-archiver/archiving.png)](https://www.percona.com/blog/2020/09/09/why-postgresql-wal-archival-is-slow/ "WAL Archiver")

So Postgres has separate archiver process... right. I tried `ps aux | grep
archiver` and voila, there is a line with
```
postgres: 13/main: archiver archiving 000000010001AFCD000000D6
```
Looks like it's stuck on this segment. For several days, as
it seems. In PostgreSQL logs we've found this:

```
INFO: 2021/11/04 12:23:29.559454 FILE PATH: 000000010001AFCD000000D5.br
INFO: 2021/11/04 12:23:30.313433 FILE PATH: 000000010001AFCD000000D7.br
INFO: 2021/11/04 12:23:30.423031 FILE PATH: 000000010001AFCD000000D8.br
INFO: 2021/11/04 12:23:30.584363 FILE PATH: 000000010001AFCD000000D6.br  # <----
INFO: 2021/11/04 12:23:31.196299 FILE PATH: 000000010001AFCD000000DA.br
INFO: 2021/11/04 12:23:31.248269 FILE PATH: 000000010001AFCD000000D9.br
INFO: 2021/11/04 12:23:31.790405 FILE PATH: 000000010001AFCD000000DB.br
# ...files in order...
INFO: 2021/11/04 12:23:39.821431 FILE PATH: 000000010001AFCD000000EB.br
INFO: 2021/11/04 12:23:40.349363 FILE PATH: 000000010001AFCD000000ED.br
ERROR: 2021/11/04 12:23:40.780994 failed to upload 'pg_wal/000000010001AFCD000000EC.br' to '*****': RequestError: send request failed
caused by: Put https://*****/*/000000010001AFCD000000EC.br: write tcp *.*.*.*:33060->*.*.*.*:443: use of closed network connection
INFO: 2021/11/04 12:23:40.781011 FILE PATH: 000000010001AFCD000000EC.br
ERROR: 2021/11/04 12:23:40.781029 Error of background uploader: upload: could not Upload 'pg_wal/000000010001AFCD000000EC'
: failed to upload '000000010001AFCD000000EC.br' to bucket '*****': RequestError: send request failed
caused by: Put https://*****/*/000000010001AFCD000000EC.br: write tcp *.*.*.*:33060->*.*.*.*:443: use of closed network connection
INFO: 2021/11/04 12:23:40.865590 FILE PATH: 000000010001AFCD000000EE.br 
# ...files in order...
INFO: 2021/11/04 12:23:45.359512 FILE PATH: 000000010001AFCD000000F7.br
# backup lines stop here
```

Time corresponds nicely with our approximate timeline of 12:30 UTC from
monitoring, where we start seeing disk usage to start trending upwards. There
are errors with `000000010001AFCD000000EC` WAL segment, and indeed we didn't have
`000000010001AFCD000000EC.br` in our backup location. Maybe Postgres archiver
would have [tried up to three
times](https://github.com/postgres/postgres/blob/REL_13_STABLE/src/backend/postmaster/pgarch.c#L67)
to re-upload it again, if it wasn't stuck on `000000010001AFCD000000D6`.

So we tried to make it unstuck. Archiver process didn't react to `kill` (that
explains why it didn't react to config change that propagates with `SIGHUP`), so
`kill -9`! It died and Postgres [launched a new
archiver](https://github.com/postgres/postgres/blob/REL_13_STABLE/src/backend/postmaster/pgarch.c#L61)
that went about deleting all the old WALs due to our
`archive_command=/bin/true`. Finally.

![Graph of disk usage](/img/pg-archiver/disk-usage.jpg "Graph of disk usage")

We have PostgreSQL 13.2 and use [wal-g](https://github.com/wal-g/wal-g) 0.22.2
as our archiver. Not sure `wal-g` version is very relevant, because it's
launched in a separate process. 

Was it a bug in [PostgreSQL
archiver](https://github.com/postgres/postgres/blob/master/src/backend/postmaster/pgarch.c)
or something else?  I had no idea how to inspect its inner state and no
knowledge about its inner workings, and wasn't ready to pour a couple of days
into figuring that out on my production server.
