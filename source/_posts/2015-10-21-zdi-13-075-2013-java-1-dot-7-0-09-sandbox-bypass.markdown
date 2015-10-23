---
layout: post
title: "ZDI-13-075 (2013) Java 1.7.0_09 Sandbox Bypass Via ConcurrentHashMap"
date: 2015-10-21 10:35
comments: true
categories: 
---

I've decided to publish the write ups for some of the Java Sandbox bypasses I disclosed to ZDI between 2012 and 2013. In total I believe
there were 19 vulnerabilities:

* ZDI-13-002
* ZDI-13-041
* ZDI-13-040
* ZDI-13-159
* ZDI-13-160
* ZDI-13-132
* ZDI-13-075
* ZDI-13-042
* ZDI-13-089
* [ZDI-13-246](/blog/2015/10/23/zdi-13-246-2013-java-1-dot-7-0-15-sandbox-bypass-via-objectoutputstream/)
* ZDI-13-079
* ZDI-13-244
* ZDI-13-245
* ZDI-13-247
* ZDI-13-248
* ZDI-13-248
* ZDI-14-105
* ZDI-14-103
* ZDI-14-104

I'm going to try and publish the more interesting ones first. I find this one involving ConcurrentHashMap
interesting because it shows how difficult Java security is to get correct. The bug was introduced by Doug Lea and
the commit introducing the bug was also reviewed by another person. If Doug Lea can't write secure code what hope
is for us mere mortals.

This bug also illustrates exploiting memory corruption by heap spraying in Java and methods to increase the reliability 
of a heap spray. Most of the vulnerabilities I found in Java did not involve heap corruption so were much more reliable.

Vulnerability
-------------

In Java 7 java.util.concurrent.ConcurrentHashMap was changed to make it go faster (?) by replacing array accesses like segments[x] to UNSAFE.getObjectVolatile(segments, SBASE + x << SHIFT).

This is the changeset from jdk source control that explains the change:

    changeset:   4021:005c0c85b0de
    user:        dl
    date:        Mon Apr 18 16:10:40 2011 +0100
    summary:     7036559: ConcurrentHashMap footprint and contention improvements

[Sun Bug](http://bugs.sun.com/view_bug.do?bug_id=7036559)

We will use the put method from ConcurrentHashMap in our vulnerability. The method looks like:

```java
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

The (hash >>> segmentShift) & segmentMask line ensures that the offset into the array is valid. Both of these values are final fields and are initialized in the constructor to the correct values. Unfortunately, this class is Serializable and the readObject method does s.defaultReadObject() and does no checking of these fields. This means an attacker can set these two values to whatever he wants. For example he can set segmentShift to 0 and segmentMask to 0xFFFFFFFF and then the value of hash will be unchanged by the (hash >>> segmentShift) & segmentMask operation. if the value of hash is larger than the array then the JDK will try to do a memory write past the array.

POC Code
--------

The POC code is available from [Github](https://github.com/benmmurphy/JavaPlayground/blob/master/ZDI-13-075/CHM.java).

In the attack we create a serialized ConcurrentHashMap with segmentShift set to 0 and segmentMask set to 0x40000. The idea behind using 0x40000 is so the calculated offset will either be 0 or 262144. When the offset is 0 the hash map works correctly and doesn't do anything weird. When the value is 262144 it will write past the end of the segments array and hopefully it will write where we want it to write. The class Make is used to generate the byte array for this object as well as the byte array for the Evil class I will cover later on.

Next we deserialize this object in the sandboxed class. We then run a gc cycle and sleep. This massively improves the chance of the exploit succeeding. I suspect this is because the array that I allocate to catch the write is more likely to appear after the ConcurrentHashMap in memory and also more likely to be closer to the ConcurrentHashMap in memory. If the segs array is > (262144 * bytes) bytes away in memory from the segments array in the concurrent hash map object then the exploit will fail and probably crash when it tries to read an object and it is not properly aligned. The exploit will also fail if the segs array is located before the ConcurrentHashMap's segment array in memory.

Next we allocate a big array of Segs to try to catch the write that ConcurrentHashMap does. The Seg class has the exact same structure as ConcurrentHashMap.Segment does. It also contains a HEntry class which has the exact same structure as ConcurrentHashMap.HashEntry. One major difference is HEntry value field is typed EvilClassLoader while ConcurrentHashMap.HashEntry value field is typed Object.

Next we call on the put method on ConcurrentHashMap. This will hopefully either write to the 0th segment in ConcurrentHashMap or write into our array of Segs.

As the value to the put call we use a ClassLoader object. The hope is that will end up with a HEntry object that has a value filled in with a ClassLoader object but which is typed EvilClassLoader. Our EvilClassLoader object has a static method which can be used to define classes with arbitrary protection domains. Normally the JVM will not let you create an EvilClassLoader object because it can subvert the sandbox. There is a check in the ClassLoader's constructor to prevent you from doing this. But if the JVM gets confused about types because of a naughty UNSAFE.putObjectVolatile then we can trick the JVM into believing a ClassLoader is really an EvilClassLoader and we can call our evil methods on the ClassLoader instance.

If we can get an object typed EvilClassLoader then it is game over because we can load the Evil class which has a method 'disable' which disables the security manager. The disable method will succeed because it does AccessController.doPrivileged and it has the AllPermission we added using the EvilClassLoader's defineClass method.

We repeat the calls to the put method with different String values because the jvm hashing algorithm is randomized and we can get unlucky and repeatedly get offset values of 0. I believe at the time Java was using Murmur/Murmur2 + mixed in secret key in order to protect against HashDoS and it was easier to just keep generating different strings than to try and reverse the secret key.


Testing
-------

    java -Djava.security.manager CHM
or

    appletviewer test.html

It will print out "user.home" system property and try to run calc.exe

I've tested this with latest jdk 1.7.0_09 on windows 7 and mac osx. If you have trouble getting it working segmentMask in Make.java (need to copy bytes to INPUT field in CHM.java after making change) and the size of segs array in CHM.java can be tweaked. 
