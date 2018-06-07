---
layout: post
title: "VoltDB Command Logging Quirks"
date: 2018-06-07 10:32:09 +0100
comments: true
categories: 
---

Since the [last post](/blog/2018/06/01/fsync-db-lock-contention/) on fsync and 
non-durable reads we have had a play around with VoltDB too see if our 
speculation with how synchronous command logging works would be consistent 
with it's performance.

The first thing we noticed is that read-only transactions wait for the log
interval even if there are no previous write transactions waiting for their
command log to be synced to disk. You can observe this by setting log frequency
to 5000ms (the maximum) and using synchronous logging. Then if you run an
ad-hoc select statement from sqlcmd you will notice that it sometimes takes
the full 5 seconds to return a result. It is important to note read-only 
transaction will not wait for a disc sync if there is only read only transactions 
in the command log buffer. But even without the sync some read only transactions
will be unecessarily be delayed because they wait for the command log to be
flushed. I have a LD_PRELOAD module that would delay 
synchronization by 1 second and I never observed a read-only transaction taking 
more than 5 seconds to return a result. However, if there are write transactions 
in the buffer then the read-only transaction will wait for the command log to be
synchronised (presumably the data from the read-only transaction isn't actually
written). This waiting for previous write transactions to sync to disk prevents 
the non-durable read problem discussed in the previous post.

It is kind of weird that VoltDB unecessarily stalls some read transactions but
this is probably not a big deal with real workloads because frequency will be 
set to 1ms and most workloads are a mix of read and writes and writes will
always cause the command log to be synchronised.

Here are some links from VoltDB that explain how command logging 
[works](https://docs.voltdb.com/UsingVoltDB/ChapCmdLog.php) and can be
[configured for optimal performance](https://docs.voltdb.com/UsingVoltDB/CmdLogConfig.php).