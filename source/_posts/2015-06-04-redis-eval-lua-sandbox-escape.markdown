---
layout: post
title: "Redis EVAL Lua Sandbox Escape"
date: 2015-06-04 09:52
comments: true
categories: 
---

It is possible to break out of the Lua sandbox in Redis and execute arbitrary
code. This vulnerability is not new and is heavily based on 
[Peter Cawley's work](https://www.youtube.com/watch?v=OSMOTDLrBCQ) with Lua bytecode 
[type confusion](https://gist.github.com/corsix/6575486).

This shouldn't pose a threat to users under the trusted Redis security model
where only trusted users can connect to the database. However, in real
deployments there could be databases that can be accessed by untrusted users.
The main deployments that are vulnerable are developers machines, places where
redis servers can be reached via SSRF attacks and cloud hosting.

# Vulnerable Deployments

## Developers Machines

Developers machines may be vulnerable because they bind Redis to all interfaces
which used to be the default listen directive in the Redis configuration.

Developers may also be vulnerable even if they bind to 127.0.0.1 because Redis
is effectively a HTTP server. The first mention of attacking Redis via HTTP I
could find is by [Nicolas Grégoire](http://www.agarri.fr/kom/archives/2014/09/11/trying_to_hack_redis_via_http_requests/index.html)
where he documents attacking a Redis server on a Facebook property using a SSRF
vulnerability.

Because Redis is a HTTP server the same origin policies of browsers will allow
any website on the internet to send a POST request to it. When using XHR the
body is completely controllable. For example if you run the following in the
console of your webbrowser while running wireshark:

    var x = new XMLHttpRequest();
    x.open("POST", "http://127.0.0.1:6379");
    x.send('eval "print(\\"hello\\")" 0\r\n');

In wireshark you will see:

    POST / HTTP/1.1
    Host: 127.0.0.1:6379
    Connection: keep-alive
    Content-Length: 27
    Pragma: no-cache
    Cache-Control: no-cache
    Origin: http://www.agarri.fr
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/42.0.2311.135 Safari/537.36
    Content-Type: text/plain;charset=UTF-8
    Accept: */*
    Referer: http://www.agarri.fr/kom/archives/2014/09/11/trying_to_hack_redis_via_http_requests/index.html
    Accept-Encoding: gzip, deflate
    Accept-Language: en-US,en;q=0.8

    eval "print(\"hello\")" 0
    -ERR unknown command 'POST'
    -ERR unknown command 'Host:'
    -ERR unknown command 'Connection:'
    -ERR unknown command 'Content-Length:'
    -ERR unknown command 'Pragma:'
    -ERR unknown command 'Cache-Control:'
    -ERR unknown command 'Origin:'
    -ERR unknown command 'User-Agent:'
    -ERR unknown command 'Content-Type:'
    -ERR unknown command 'Accept:'
    -ERR unknown command 'Referer:'
    -ERR unknown command 'Accept-Encoding:'
    -ERR unknown command 'Accept-Language:'
    $-1

And in the stdout for Redis you will see:

    hello

The attacker is not able to read the response from the server because of the
same origin policy. However, this might be worked around by using a [DNS rebinding attack](https://miki.it/blog/2015/4/20/the-power-of-dns-rebinding-stealing-wifi-passwords-with-a-website/).
Even with DNS rebinding it might not be possible to read the response because
the response is not valid HTTP.

However, reading the response is not necessary because you can package a super generic
exploit that checks the result of the redis.call("INFO") command and then
launches a OS/architecture specific payload.

## SSRF attacks

This is similar to attacking developers except a trusted server is tricked into
making a request to the Redis server However, you need a lot of control over
the body which might not often be possible depending on how the body is encoded.

## Redis Cloud Hosting

This will only effect providers where people running arbitrary code from the
Redis process is not part of their threat model. The major players in this area
look like they are using sandboxing. For example the pids returned by 'INFO' on
heroku are very low <10 which indicates they are running the Redis servers in
containers. You can already run arbitrary code in containers via dynos on Heroku
so running arbitrary code in a Redis container is probably not useful for an
attacker. Amazon Elasticcache also looks like it uses linux containers.

Similarily, it looks like Microsoft's hosted Redis solution runs in an
isolated VM. Redis 'INFO' returns a virtual os string and it takes ~15
minutes to launch an instance. If MS aren't running in an isolated VM then the
15 minute startup time is very weird.

This will be a problem if a hosting provider runs a whole bunch of redis
processes on the same machine/same VM from different customers without any
kind of isolation.

# Exploit

[Peter Cawley](https://gist.github.com/corsix/6575486) has found that the 
loadstring function can be used to load bytecode that is unsafe. He has 
created three very useful lua exploit primitives that make exploitation easy.

First is a way of reading the Value contained in a TValue struct. This allows
reading the pointer value from a lua tagged value. Some pointer values are
already public (using tostring) but there doesn't seem to be a way to get the
pointer value for a lua string so this is useful.

Second is a way of reading 8 bytes from an arbitrary memory address.

Third is a way of writing 8 bytes to an arbitrary memory address.

Using the arbitrary memory read it is possible to leak the address of a known
C function. From the address of this c-function it is possible to find the base
address of the redis-server binary. From this base address it is possible to
find pointers to libc/libsystem_c functions and to find the base address of the
libc/libsystem_c libraries. From these libraries it is possible to find the
addresses of useful exported functions (longjump/system) and ROP gadgets. This
technique is similar to pwntools [dynelf](http://pwntools.readthedocs.org/en/latest/dynelf.html)

The arbitrary memory read is also used to leak an address inside the stack. The
lua_State object holds a long_jump variable that references a long_jump buffer
that is allocated on the stack. This leaks the stack address which can be useful
if you just want to corrupt the stack or the rsp and rip can be overwritten in
the longjump buf to directly take control when longjump is called. OSX has no
pointer mangling protections so this is quite easy to corrupt.

On linux the rip and rsp (and rbp) values are mangled. However, if you have full
read access to the memory you can reverse the secret cookie value to corrupt
the values. Also, linux prevents you from longjmp'ing to an invalid stack frame
(ie: the heap) but you can longjump to point the stack inside the longjump
buffer then pivot the stack into the heap. This is not really necessary if you
don't care about corrupting the stack and crashing the redis process but if you
longjump and don't corrupt the stack then you can resume normal execution of
redis after the exploit has finished running.

# Exploitability

I have exploits for Linux 64 bit and OSX 64 bit. Both exploits take care to not
crash the redis server during successful execution. They will make a call to
system() then go back to normal redis execution.

I have run the Linux exploit on the Amazon RHEL Image (PIE enabled) and the
Amazon 14.04 Ubuntu Image (no PIE). I believe the exploit will work on most
modern Linux 64 bit systems (I suspect it will not work if you compile libc with
 fomit-frame-pointer but this can be worked around). It does not use any
hardcoded addresses from libc or the Redis binary.

The OSX version I have only tested on Yosemite but an earlier version was
working on Mavericks and I believe the Yosemite version works on both. This has
been tested with two different Redis versions and similarily does not depend on
hardcoded address from libsystem_c or the Redis binary. However, it uses
addresses from libsystem_c to speed up the exploit.

# Workarounds

The best option is to set a strong password on Redis. Systems that are reachable
via HTTP without a password are a problem waiting to happen.

It is also possible to rename the EVAL command. If you are not using EVAL this
is a good option but you still might be at risk of someone modifying your Redis
data via HTTP SSRF attacks.
