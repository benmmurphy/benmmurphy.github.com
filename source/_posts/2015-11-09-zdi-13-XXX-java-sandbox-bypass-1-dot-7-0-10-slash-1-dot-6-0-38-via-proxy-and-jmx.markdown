---
layout: post
title: "ZDI-13-XXX (2013) Java Sandbox Bypass (1.7.0_10) / (1.6.0_38) via Proxy and JMX"
date: 2015-11-09 10:41
comments: true
categories: 
---

This is part of a series of posts detailing Java Sandbox Bypasses that were disclosed
between 2012-2013. You can view the other bugs by going back to the [original post](/blog/2015/10/21/zdi-13-075-2013-java-1-dot-7-0-09-sandbox-bypass).

The last two vulnerabilities I wrote up ( [ZDI-13-246](/blog/2015/10/23/zdi-13-246-2013-java-1-dot-7-0-15-sandbox-bypass-via-objectoutputstream), [ZDI-13-075](blog/2015/10/21/zdi-13-075-2013-java-1-dot-7-0-09-sandbox-bypass)) involved heap spraying so were not 100% reliable. Most of my sandbox bypasses against the JVM did not use memory corruption or heap spraying so were 100% reliable. These reliable sandbox bypasses fell into two main categories:

First there were vulnerabilites that would try to create a chain from privileged code to a 'dangerous' function without touching any user frames. Java uses stack walking to decide whether a dangerous function (`System.setSecurityManager(null)`, `Runtime.execute`) is allowed to proceed so if you could create a chain then you could subvert the protection. 

Second there were vulnerabilities that got access to methods in the 'protected packages'. After getting access to these packages it is usually trivial to escalate out of the sandbox because it is assumed user code cannot access these methods. Access to these packages usually involved abusing reflection or parts of the JDK that used reflection but did not do so securely. This vulnerability which has existed at least since Java 5 is a good example of abusing reflection to access privileged packages. 


This bug is interesting because there is no ZDI public disclosure for it. I suspect this is because [Adam Gowdiak](http://www.security-explorations.com/en/about.html) disclosed it to Oracle first. Looking back I also suspect I may have sniped this vulnerability from Adam Gowdiak. Gowdiak seems to have a habit of partially publicly disclosing Java bugs before they are fixed. Another bug I disclosed to ZDI, ZDI-13-079 was based on a post he made to the full disclosure mailing list and I definitely sniped this bug from him. I can't remember the exact details about how I found this bug but I remember Gowdiak made a presentation where he said 'com.sun.xml.internal.bind.v2.model.nav.Navigator' was an interesting class. It is possible that I was able to reverse the underlying bug from this. 

Vulnerabilies
-------------

Three vulnerabilities are used to bypass the sandbox.

1. Accessing Class instances in protected packages.
2. Reading fields on interfaces in protected packages.
3. Getting access to `java.lang.reflect.Method` for interface methods in
protected packages.


Loading Classes in Protected Packages
-------------------------------------

The JmxMBeanServer class allows you to load classes from protected packages.
This isn't possible in Java 6.

    server = JmxMBeanServer.newMBeanServer("", null, null, true);
    server.getMBeanInstantiator().findClass(className, (ClassLoader)null);

findClass in MBeanInstantiator ends up calling `loadClass(className, null)`
which will end up performing `Class.forName(className)`.



```java MBeanInstantiator.loadClass http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/com/sun/jmx/mbeanserver/MBeanInstantiator.java#MBeanInstantiator.loadClass%28java.lang.String%2Cjava.lang.ClassLoader%29
    static Class<?> loadClass(String className, ClassLoader loader)
        throws ReflectionException {

        Class<?> theClass;
        if (className == null) {
            throw new RuntimeOperationsException(new
                IllegalArgumentException("The class name cannot be null"),
                              "Exception occurred during object instantiation");
        }
        try {
            if (loader == null)
                loader = MBeanInstantiator.class.getClassLoader();
            if (loader != null) {
                theClass = Class.forName(className, false, loader);
            } else {
                theClass = Class.forName(className);
            }
        } catch (ClassNotFoundException e) {
            throw new ReflectionException(e,
            "The MBean class could not be loaded");
        }
        return theClass;
    }
```

Reading Fields on Interfaces in Protected Packages
---------------------------------------------------

If you call `Proxy.getProxyClass(null, new Class[]{targetClass})` then the
generated proxy class will have all the fields from the targetClass. Because
the generated proxy class is not in a protected package user code can then call
`proxyClass.getFields()` which will give back the `java.lang.reflect.Field` object
and because the field is public call `Field#get` will succeed. The proxy class
successfully loads because it is defined the root class loader.

Getting Access to Method objects for Interface Methods in Protected Packages
----------------------------------------------------------------------------

This uses a similar vulnerability as above. You can think of the Proxy instance
as a machine that will convert Method objects into Method objects for a
particular interface. If you look at `proxyClass.getDeclaredMethods()` for
`com.sun.xml.internal.bind.v2.model.nav.Navigator` you will see something like:

    public final boolean $Proxy0.isFinal(java.lang.Object)
    public final boolean $Proxy0.isArray(java.lang.Object)
    ..

If you call `$Proxy0.isFinal(java.lang.Object)` then it will convert this Method
into `Navigator.isFinal(java.lang.Object)` before passing it to the
`InvocationHandler`.

To access a `Method` on an interface in a protected package all you have to do is
create an `InvocationHandler` that will save the Method then invoke the
corresponding public method on the proxy class.

Once an attacker has access to the Method then they are free to invoke it
because the `Method` is public and no more access checks are performed.

Exploit
-------

1. We use the JMX class loading vulnerability to load the class
`"com.sun.xml.internal.bind.v2.model.nav.Navigator"`.
2. We then use the field reading vulnerability to read the `REFLECTION` field from
the interface.
3. We then use the interface method vulnerability to read the
`getDeclaredMethods(Object o)` method from the `Navigator` class.

Now that we have a way of getting Methods from a protected Class
(`getDeclaredMethods`) and a way of loading protected classes (JMX vulnerability)
we can easily subvert the JVM sandbox. There is probably 100 ways of doing this
because once you can execute arbitrary static methods in the protected packages
it is game over for the JVM. We will use a technique similar to the one
disclosed in ZDI-13-159 in order to disable the sandbox except we will modify
it slightly so it only uses JDK 6 classes.

1. We use `com.sun.xml.internal.bind.v2.ClassFactory#create(Class)` to create a
`sun.reflect.ReflectionFactory$GetReflectionFactoryAction`
2. We use `com.sun.xml.internal.ws.api.server.InstanceResolver#createSingleton` to
create an `InstanceResolver` object
3. We use `com.sun.xml.internal.ws.api.server.InstanceResolver#createInvoker` to
create an `Invoker` object
4. We use `com.sun.xml.internal.ws.api.server.Invoker#invoke` to invoke
`AccessController#doPrivileged` with the `PrivilegedAction` in step 1 to create a
`ReflectionFactory` object.
5. We invoke `sun.reflect.ReflectionFactory#newField` with parameters that
correspond to the `Statement#acc` field
6. We invoke `sun.reflect.ReflectionFactory#newFieldAccessor` with the new field
object.
7. We create a `Statement` object that executes `System.setSecurityManager(null)`;
8. We invoke `sun.reflect.FieldAccessor#set(Object, Object)` with a `Statement`
object we have created and a `AccessControlContext` that gives us all permissions
9. We execute the `Statement` which disables the JVM security.

Exploit Java 6
--------------

We use the same technique as above but we use the XSLT class loading hack
disclosed in ZDI-13-159 to load the classes because this works in Java 6.

Testing (Java 7)
----------------
The POC is available from [Github](https://github.com/benmmurphy/JavaPlayground/tree/master/ZDI-13-XXX/proxy_abuse7)

java -Djava.security.manager ProxyAbuse
or
appletviewer test.html

It will try and print the users home directory and execute an apple script that
will say some stuff.

Testing (Java 6)
----------------
The POC is available from [Github](https://github.com/benmmurphy/JavaPlayground/tree/master/ZDI-13-XXX/proxy_abuse6)

java -Djava.security.manager Harness
or
appletviewer test.html

It will try and print the users home directory and execute an apple script that
will say some stuff.

Fixes
-----

User code probably shouldn't be able to load Proxy Classes in the bootstrap
class loader.

