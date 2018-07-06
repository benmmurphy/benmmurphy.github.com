---
layout: post
title: "Postgresql Non-Durable Reads"
date: 2018-07-06 14:29:46 +1000
comments: true
categories: 
---

This is an experiment where we try to find out how much faster Postgres will run 
if patch it to support non-durable reads. Postgresql supports a number of
options when commiting but the ones we are interested in at the moment are
`synchronous_commit = on` and `synchronous_commit = off`. When it is set to `on`
Postgresql will wait until the WAL is properly synchronized to the disk before
returning to the client and when it is `off` it will not wait at all. Also,
when it is `on` postgresql will not release any locks or make any tuples visible
to other transactions until the WAL is synchronized to disk. This means if you
have heavy write contention on a particular row then all of your connections
can end up being serialized around fsync().

For example take this query:

```
UPDATE counters set counter = counter + 1 where id = 0;
```

If you run this using pg_bench and 128 clients on Macbook Air with 
`wal_sync_method = fsync` and `synchronous_commit = on` you will get around
1283 transaction/s. This is with an fsync cost of about 0.2 ms. If you
change `wal_sync_method = fsync_writethrough` this increases the fsync cost
to about 10ms and the throughput will drop to around 90 transaction/s. If you
switch `synchronous_commit = off` this will increase perforamnce and you will 
get around 2611 transaction/s. However, this is kind of unsafe and you might 
lose transactions that you think have been committed if you have a sudden loss 
of power.

Now, we have made a change to postgresql to drop its locks and record the 
transaction as visible and then wait for the WAL log to be synchronised to disk.
This gives you similar guarantees to `synchronous_commit = on` and similar
performance to `synchronous_commit = off` when there is high contention. When,
there is low contention it has similar performance to `synchronous_commit = on`.

After, this change we get 3321 transactions/s which is about 2.5 times better
performance that `synchronous_commit = on` and suspiciously even higher
than `synchronous_commit = off`. Currently, I don't have an explanation for this
and it would seem to indicate that there is an implementation error in the
patch. (Definitely, do no trust this patch with real data this is currently just
an experiment.) I would expect that this new mode would perform strictly worse 
than `synchronous_commit = off`.

It also important to note that using this setting 
`sychronous_commit = local_non_durable_reads` will produce read anomalies that
can be similar to the write anomalies you get with `sychronous_commit = off`. 
For example if you insert an tuple and it fails with a unique check then you
later try to read that tuple then it is possible it won't be there anymore
because the original write was not durable when the read was performed for the
unique check. But for this counter example it is 100% safe as long as you never
try to read the value of the counter :)

If there is any interest in this I might try to upstream the patch into PG. I'm
just not sure if there are people running with high enough contention and high
enough fsync times to make this setting useful.

[NonDurableReadPatch](https://github.com/postgres/postgres/compare/master...benmmurphy:non_durable_reads)
[Benchmarks](https://gist.github.com/benmmurphy/49e83da3f8b0793627e4a02e0d154c82)

