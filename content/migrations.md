+++
title = "ORM database migration tools"
date = 2020-05-04
+++

This is a rant mainly about ORM-based migration tools for SQL. 

Why SQL exactly?  I haven't touched MongoDB world for almost a decade now (what
a relief). Cassandra never really crossed my way. Any sane team has its own
Elasticsearch migration tool that's called "create all indices from scratch",
nothing interesting to say here. The only other database I've ever touched is
Datomic, and they have [pretty interesting
things](https://blog.datomic.com/2017/01/the-ten-rules-of-schema-growth.html) to
say about migrations that I won't touch on.

Anyway, to the topic. Why do ORM-based migration tools exist? Lets look at a
couple of projects own descriptions for a clue:

> **[Django](https://docs.djangoproject.com/en/3.0/topics/migrations/)**:
> Migrations are Django’s way of propagating changes you make to your models
> (adding a field, deleting a model, etc.) into your database schema. They’re
> designed to be mostly automatic, but you’ll need to know when to make
> migrations, when to run them, and the common problems you might run into.

> **[Ruby on
> Rails](https://guides.rubyonrails.org/active_record_migrations.html#migration-overview)**:
> Migrations are a convenient way to alter your database schema over time in a
> consistent and easy way. They use a Ruby DSL so that you don't have to write
> SQL by hand, allowing your schema and changes to be database independent.

Looks like there are two purposes for these tools: 

* Database independent migrations
* Automatic and easy migartions when possible

I've personally used Django migrations (and when it was a separate tool called
South) and skimmed docs for some other tools: Alembic, RoR migrations,
Sequelize, TypeORM.

Database independent migrations
--------

Database independence is certainly useful for some Django-style "apps". They're
essentialy libraries that define their own models and can update them in new
versions. User management libraries come to mind, like
[django-allauth](https://github.com/pennersr/django-allauth) or
[python-social-auth](https://github.com/python-social-auth/). Without built-in
migrations we'd have to read their changelogs and apply schema changes by hand
like cavemen.

I am now of opinion that any Django-style "app" is only useful for short
projects that are put together in a couple of months. In the long-running
projects they usually get in the way much more than they help.

The vast majority of web applications and products are run with one database
ever and don't switch from Oracle to MySQL to PostgreSQL and back.

Automatic and easy migartions 
-------

These tools strive to generate migrations automatically. The process goes like
this: read the documentation, add a field to a model declaration, run some
command and boom! We've got our migration and didn't have to write `ALTER TABLE
table CREATE COLUMN` now, so I traded a minute of writing a line for a solid
hour or two of reading docs and poking this new tool. Fascinating experience,
I hope it will at least save us more time in the future.

One week later I rename some field or table, and now the tool wants to
delete the field and create another. Guess I have to go back and read more
documentation to remember how to instruct a tool to actually rename a field
instead of dropping it and creating a new one. Do I really want to avoid writing
`ALTER TABLE` that much?

Weeks pass, I get more experienced with a tool, remember the command and what
flags to pass, I learned how to rename this and that, spent some time debugging
my migrations. Hours and hours on top of simple `ALTER TABLE`.

### Non-trivial migrations

Now I need to do some non-trivial migration on a big database. For example, it's
a combination of creating a couple of fields, filling them with information,
creating additional indexes, dropping old fields and indexes. In such cases I
usually open postgres shell to a local DB, open a transaction and then I develop
a migration like a code in a REPL. `BEGIN`, then create some columns, fill them
with info, SELECT to check that it's all right. Nope, messed up some CASE in
UPDATE and dropped like third of relevant information. Not a problem,
`ROLLBACK`, paste the working part and try again.

In case of these automatic tools, after I did all that, I have to go read
their quite massive documentation again, because I forgot some things, and port
SQL to their syntax. Or should I just drop into raw SQL now? What was the point
of using the tool from the start?

### Onboarding new developers

Then we added junior members to the team who didn't yet know SQL well. Easier
tasks are "automated" by the tool, they only had to read a documentation for a
couple of hours instead of learning how to write an actual `ALTER TABLE` in SQL,
that it can be rolled back inside a transaction, etc.

And now, when juniors are tasked with a bit more complex problem, they
completely lack skills developing a migration and have a much steeper wall to
climb. So they sit in front of their computers, staring into documentation for
hours for what should have been an easy task for now.

### Downgrade migrations

Another thing is that many tools have downgrade migrations. Downgrade migrations
are a lie! How could I revert a migration that drops a NOT NULL column with some
data? In case I really need to revert a migration on production I will write
another forward migration. Which I did exactly zero times in more than ten
years.

During development I can switch branches that have different schemas, and they
can be incompatible. Of course, then I have to go and do a backward migration
with ALTER and even delete a line from tracking table. I did that a couple of
times in the last five years. Could pervasive downgrade migrations save me these
ten minutes? Doubt it, often such incompatibility stems from irreversible
changes like dropping NOT NULL column and migration tools will just spew an
`IrreversibleError`.

So, writing downgrade migrations is another waste of time.

### Operations nitpicking

Another minor point is that sometimes migrations run for a loooooong time. Not a
second or two - it can be many hours on production. I don't want this migration
to start automatically during my deploy process (my CI will think it died, among
other things), but it should be a regular part of all migrations, so on local
developers' installations these migrations just take a usual `make migrate`
route. It's much more convenient to take a part of a SQL script and run it
separately, than to take a part of a migration script written in Python and run
it separately.

### Conclusions

When I inherited a Django project with working migrations several years ago,
I've used this madness for a couple of months and then switched back to a simple
SQL-based system.

While these tools "automate" simplest cases, they make more complex cases much
harder. Sadly, it's a very common pitfall in Django-land, and I heard that RoR
is pretty much the same.

I'm personally using a simple tool called
[Nomad](https://pypi.org/project/nomad/). I think any tool that supports plain
SQL migrations, has dependencies between migrations and doesn't require to write
downgrades would be acceptable.

Less is so much more in this case.
