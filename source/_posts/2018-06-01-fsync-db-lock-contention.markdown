---
layout: post
title: "FSync DB Lock Contention"
date: 2018-06-01 19:46:30 +0100
comments: true
categories: 
---

Lets say you have a table in a database that you are using to keep track of a 
counter value. So something like:

{% codeblock %}
 Column |  Type   | Modifiers
--------+---------+-----------
 id     | integer | not null
 count  | integer | not null
Indexes:
    "test_pkey" PRIMARY KEY, btree (id)
{% endcodeblock %}

If you have lots of connections updating values doing the query: ``` update 
test set count = count + 1 where id = $1;``` then on postgresql you should have
very decent performance as long as the id values do not overlap too much. Even
though postgresql needs to fsync the WAL to disk for each commit it is able to
amortise the cost of this over many commits if the commits start to queue up
because fsyncing is slow. For example the WAL fsyncs might be something like:

{% codeblock %}
update test set count = count + 1 where id = 1
FSYNC WAL
update test set count = count + 1 where id = 2
update test set count = count + 1 where id = 3
..
update test set count = count + 1 where id = 100
FSYNC WAL
update test set count = count + 1 where id = 101
update test set count = count + 1 where id = 102
..
update test set count = count + 1 where id = 200
FSYNC WAL
{% endcodeblock %}

However, if the id values overlap and in the worst case if they are all the same
then not only do you have a problem with lock contention but you also have a
problem with serializing all of the fsyncs. Postgresql and presumably any sane
RDBMS will hold a write lock on the updated records until the transaction is
durable. So you end up getting all the WAL fsyncs done in a completely serial
order.

{% codeblock %}
update test set count = count + 1 where id = 1
FSYNC WAL
update test set count = count + 1 where id = 1
FSYNC WAL
update test set count = count + 1 where id = 1
FSYNC WAL
update test set count = count + 1 where id = 1
FSYNC WAL
{% endcodeblock %}

Before the land of SSDs this would be absolutely horrible. If you are paying
6ms for a fsync then it completely destroys your throughput (166 fsync/s). But 
now with SSDs (or previously with battery backed caches) the fsync cost is much 
lower so this is less of an issue. For example with Amazon EBS I see fsync cost 
of around 0.5ms (2000 fsync/s) and i3 NVME performance of ~0.05ms 
(20000 fsync/s). 

Is it possible for an RDBMs to fix this fsync problem? So when you think about
it an RDBMs could drop all the locks a transaction has once it has decided that
nothing except for the WAL fsync would prevent it from committing. This would
kind of work because another transaction that would modify the same row would
be dependent on the previous WAL segments committing before it could commit. 
However, this opens up a big consistency hole in the way clients interact 
with the database. For example you could see this happening:

{% codeblock %}
TX1: BEGIN;
TX1: insert into test(id) values(1);
TX1: COMMIT;

<TX1 not fsync'd>

TX2: BEGIN;
TX2: insert into test(id) values(1);

<instead of blocking here for TX1 to commit it raises unique error>

DB: POWER FAILURE <TX1 is never committed>

{% endcodeblock %}

Here we see a case where the second transaction observes data in the database
that was not durable. It might think that because the record is already in
the database it can do something else with an external system and then we end
up having a problem. This particular case is also weird because the transaction
gets in a state where it can't be committed. If you successfully commit a 
transaction that has touched non-durable records then all the reads are safe
because the records would now be durable after the commit. But a transaction
with an error is not committable so you would also need to add a weird hook
where a rollback (or implicit rollback) might have to wait for other
transactions to fsync before returning to the client.

Also, transactions that did not modify data would normally have a no-op commit
but if they were shown non-durable records they would need to potentially wait
to commit.

Basically, it could kind of work as long as all transactions waited for a 
successful commit/rollback before acting on any data they read from the DB 
but this does not seem realistic.

If you look at VoltDB it looks like they only let you do transactions inside
stored procedures. I've also seen a comment along the lines that they do batch
commits. Considering, they are single threaded presumably they handle a bunch
of transactions on the thread and add them to a queue that is fsynced in batches
then the results from the stored procedures are sent back to the client. This 
presumably removes any of the consistency problems you have from a system 
that has external transactions where non-durable reads can escape.

If you want to play around with postgresql to see what effects wal fsync delay
has I have a LD_PRELOAD library that will add 10s to every fsync. 
[wal_delayer](https://github.com/benmmurphy/wal_delayer)

