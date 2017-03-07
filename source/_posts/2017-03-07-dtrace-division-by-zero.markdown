---
layout: post
title: "DTrace Division by Zero"
date: 2017-03-07 09:55:03 +0000
comments: true
categories: 
---

For some background check John Regehr's [excellent post](http://blog.regehr.org/archives/887) on the history of problems caused by dividing INT_MIN by -1. DTrace is an interpreter that runs inside the kernel on both Illumos and OSX. Before it was [patched](https://github.com/joyent/illumos-joyent/commit/8a5ff7873220bd2725876b6ef7fdd2bceff60dd3) in Illumos it was possible to create an expression to divide INT_MIN by -1 and this would cause the kernel to crash. 

```
sudo dtrace -n 'BEGIN{v = 0x8000000000000000LL; print((long)v/-1)}'
sudo dtrace -n 'BEGIN{v = 0x8000000000000000LL; print((long)v%-1)}'
```

This is still an issue in OSX. I emailed them a month ago along with links to other DTrace issues that have been fixed in Illumos and not OSX and haven't heard back. Since this is not really a security issue I'm posting it here. You need root in order to trigger the DTrace division by zero and if you have root you can already reboot the machine :/. You also need root to trigger all of the other issues.

