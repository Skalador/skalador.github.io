---
title: Who rebooted my VM? A cloud mystery
description: Investigating from a linux perspective, if the cloud provider is to be blamed for a virtual machine reboot.
slug: who-rebooted-my-vm
date: 2025-04-04 00:00:00+0000
image: cover.png
categories:
    - kubernetes
    - openshift
    - linux
tags:
    - kubernetes
    - openshift
    - linux
    - cloud
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## Who rebooted my VM? A cloud mystery

I had been recently involved in an interesting discussion on how to figure out, who has rebooted your virtual machine from a linux perspective. In this particular case, we were talking about an OpenShift `node`. I will be elaborating on a few different cases and how those can be investigated.

To reproduce the problem, I am using a five node OpenShift cluster which is running on a `Kernel Based Virtual Machine (KVM)` host.
```sh
$ oc get nodes
NAME        STATUS   ROLES                         AGE     VERSION
compute-0   Ready    worker                        3d23h   v1.31.6
compute-1   Ready    worker                        3d23h   v1.31.6
master-0    Ready    control-plane,master,worker   3d23h   v1.31.6
master-1    Ready    control-plane,master,worker   3d23h   v1.31.6
master-2    Ready    control-plane,master,worker   3d23h   v1.31.6

$ virsh list
 Id   Name             State
--------------------------------
 2    ocp4-master-0    running
 3    ocp4-master-1    running
 4    ocp4-master-2    running
 5    ocp4-compute-0   running
 6    ocp4-compute-1   running
```

Consequently, I am my own hypervisor admin which will be sabotaging my OpenShift cluster for this investigation.

What all cases will have in common is that a node will be `NotReady` for some time, e.g.:
```sh
$ oc get nodes   
NAME        STATUS     ROLES                         AGE     VERSION
compute-0   Ready      worker                        4d15h   v1.31.6
compute-1   NotReady   worker                        4d15h   v1.31.6
master-0    Ready      control-plane,master,worker   4d15h   v1.31.6
master-1    Ready      control-plane,master,worker   4d15h   v1.31.6
master-2    Ready      control-plane,master,worker   4d15h   v1.31.6
```

Additionally you will be able to find logs of the machine config controller which reports a `NotReady` machine:
```sh
$ oc -n openshift-machine-config-operator logs machine-config-controller-5f88fbbc4f-rqjm4 -c machine-config-controller | grep -i compute-1
...
I0404 06:41:13.153952       1 node_controller.go:584] Pool worker: node compute-1: Reporting unready: node compute-1 is reporting OutOfDisk=Unknown
I0404 06:41:13.196590       1 node_controller.go:584] Pool worker: node compute-1: changed taints
I0404 06:41:19.481728       1 node_controller.go:584] Pool worker: node compute-1: changed taints
I0404 06:42:42.730148       1 node_controller.go:584] Pool worker: node compute-1: Reporting unready: node compute-1 is reporting NotReady=False
...
I0404 06:42:56.309260       1 node_controller.go:584] Pool worker: node compute-1: Reporting ready
```


## Case 1: User reboot

In this case a user has rebooted one of your nodes (either via a `debug` container or via `ssh`). This can be easily found with the audit logs:
```sh
$ cat /var/log/audit/audit.log | grep reboot
...
type=USER_CMD msg=audit(1743748835.979:805): pid=1366614 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:spc_t:s0 msg='cwd="/" cmd="reboot" exe="/usr/bin/sudo" terminal=? res=success'UID="root" AUID="unset"
```

## Case 2: Machine Config rollout

A machine config rollout can also be found in the `audit.log` files:
```sh
$ cat /var/log/audit/audit.log | grep reboot
...
type=SERVICE_START msg=audit(1743789169.017:1277): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=machine-config-daemon-reboot comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'^]UID="root" AUID="unset"
type=SERVICE_STOP msg=audit(1743789169.119:1278): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=machine-config-daemon-reboot comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'^]UID="root" AUID="unset
```

Furthermore this can be confirmed with the `MachineConfigPool`, which provides us with the latest `MachineConfig` `rendered-worker-04636f9acf052beffabf57725a564095`
```sh
$ oc get mcp worker       
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
worker   rendered-worker-04636f9acf052beffabf57725a564095   True      False      False      2              2                   2                     0                      5d2h
```

Looking at the boot IDs, we should see, that boot id `57cdb3ac10fd46c3933a3f0ba3e3c33f` should be shutdown by the machine config operator with a message, that it will boot into `rendered-worker-04636f9acf052beffabf57725a564095`. This is the case:
```sh
$ journalctl --list-boots | tail -n 2
 -1 57cdb3ac10fd46c3933a3f0ba3e3c33f Fri 2025-04-04 07:47:43 UTC Fri 2025-04-04 17:53:13 UTC
  0 6004e8be8fbc4d769485a991ff1641d4 Fri 2025-04-04 17:53:19 UTC Fri 2025-04-04 18:04:01 UTC
```
```sh
$ journalctl -u machine-config-daemon-reboot
...
-- Boot 57cdb3ac10fd46c3933a3f0ba3e3c33f --
Apr 04 17:52:49 compute-1 systemd[1]: Started machine-config-daemon: Node will reboot into config rendered-worker-04636f9acf052beffabf57725a564095.
Apr 04 17:52:49 compute-1 systemd[1]: machine-config-daemon-reboot.service: Deactivated successfully.
Apr 04 17:52:49 compute-1 systemd[1]: Stopped machine-config-daemon: Node will reboot into config rendered-worker-04636f9acf052beffabf57725a564095.
```



## Case 3: A container with elevated privileges was compromised

Some containers require elevated privileges to perform certain actions on OpenShift. If such a container is compromised, then a reboot could be initiated from the container. In this example I will be using a container of the `Node Tuning Operator`:
```sh
$ oc -n openshift-cluster-node-tuning-operator get pods -owide                                          
NAME                                            READY   STATUS    RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
cluster-node-tuning-operator-5bb647f57c-lk7wx   1/1     Running   0          4d15h   X.X.X.X         master-1    <none>           <none>
tuned-5q7vm                                     1/1     Running   1          4d15h   X.X.X.X         compute-0   <none>           <none>
tuned-8n89v                                     1/1     Running   1          4d15h   X.X.X.X         compute-1   <none>           <none>
tuned-g7nwk                                     1/1     Running   0          4d15h   X.X.X.X         master-0    <none>           <none>
tuned-rxt2q                                     1/1     Running   0          4d15h   X.X.X.X         master-2    <none>           <none>
tuned-xsm98                                     1/1     Running   0          4d15h   X.X.X.X         master-1    <none>           <none>
$ oc -n openshift-cluster-node-tuning-operator rsh tuned-8n89v
$ id
uid=0(root) gid=0(root) groups=0(root)
$ date
Fri Apr  4 07:25:01 UTC 2025
$ reboot           
```

One interesting fact here is, that we do not see this reboot in the `audit.log`, as the last entry for a `reboot` is the same as the one from `Case 1`:
```sh
$ cat /var/log/audit/audit.log | grep reboot | tail -n 1
type=USER_CMD msg=audit(1743748835.979:805): pid=1366614 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:spc_t:s0 msg='cwd="/" cmd="reboot" exe="/usr/bin/sudo" terminal=? res=success'UID="root" AUID="unset"
```

Nevertheless, we can see in the `journalctl` that the `reboot` was initiated. Please also note, that the reboot was initiated shortly after `Fri Apr  4 07:25:01 UTC 2025`:
```sh
$ journalctl -b -1 | grep -i reboot | tail -n 3
Apr 04 07:25:06 compute-1 systemd-logind[906]: The system will reboot now!
Apr 04 07:25:06 compute-1 systemd-logind[906]: System is rebooting.
Apr 04 07:26:41 compute-1 systemd[1]: Requested transaction contradicts existing jobs: Transaction for NetworkManager-dispatcher.service/start is destructive (reboot.target has 'start' job queued, but 'stop' is included in transaction).
```

While an audit rule might be helping to detect that the `reboot` was started via `/usr/sbin/reboot` from a container, this is probably not something of practical relevance.

## Case 4: The hypervisor triggered a soft reboot

To differentiate better between those cases, we will be using `compute-0` for the reboots initiated by the hypervisor / cloud provider instead of `compute-1`.
For a (kind of) graceful shutdown we will be using the `virsh reboot` command. To provide a rough idea what this does, we can take a look at the man page for `virsh reboot`:
> Reboot a domain.  This acts just as if the domain had the reboot command run from the console.  The command returns as soon as it has executed the reboot  action, which may be significantly before the domain actually reboots.

```sh
$ date
Fri Apr  4 06:59:52 PM CEST 2025

$ virsh reboot ocp4-compute-0
Domain 'ocp4-compute-0' is being rebooted
```

We can find a related `audit.log`:
```sh
$ ausearch -m SERVICE_STOP | grep -Ei 'shutdownd|shutdown|reboot|final|auditd'
type=SERVICE_STOP msg=audit(1743785996.839:279): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=dracut-shutdown comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
```

Additionally we see the shutdown process in the `journalctl` of the previous boot:
```sh
$ journalctl -b -1 | grep -i shutdown | tail -n 6
Apr 04 16:59:56 compute-0 systemd[1]: Stopping Restore /run/initramfs on shutdown...
Apr 04 16:59:56 compute-0 systemd[1]: dracut-shutdown.service: Deactivated successfully.
Apr 04 16:59:56 compute-0 systemd[1]: Stopped Restore /run/initramfs on shutdown.
Apr 04 17:00:42 compute-0 systemd[1]: Requested transaction contradicts existing jobs: Transaction for NetworkManager-dispatcher.service/start is destructive (shutdown.target has 'start' job queued, but 'stop' is included in transaction).
Apr 04 17:00:43 compute-0 systemd[1]: Stopping Record System Boot/Shutdown in UTMP...
Apr 04 17:00:43 compute-0 systemd[1]: Stopped Record System Boot/Shutdown in UTMP.
```
Note: While the timezone configuration is different, those logs are related to the same `virsh reboot`.

## Case 5: The hypervisor triggered a hard reset

For this experiment we will be using the `virsh reset` command. To provide a rough idea what this does, we can take a look at the man page for `virsh reset`:
> Reset  a domain immediately without any guest shutdown. reset emulates the power reset button on a machine, where all guest hardware sees the RST line set and reinitializes internal state.
> Note: Reset without any guest OS shutdown risks data loss.

```sh
$ date
Fri Apr  4 07:17:45 PM CEST 2025

$ virsh reset ocp4-compute-0
Domain 'ocp4-compute-0' was reset
```

The only `audit.log` we can find, is the one form before:
```sh
$  ausearch -m SERVICE_STOP | grep -Ei 'shutdownd|shutdown|reboot|final|auditd'
type=SERVICE_STOP msg=audit(1743785996.839:279): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=dracut-shutdown comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
```

The `journalctl` of the previous boot also suddenly ends without a shutdown sequence:
```sh
$ journalctl -b -1 |  tail -n 2
Apr 04 17:16:11 compute-0 systemd[1]: Finished Cleanup of Temporary Directories.
Apr 04 17:16:11 compute-0 systemd[1]: run-credentials-systemd\x2dtmpfiles\x2dclean.service.mount: Deactivated successfully.
```
Note: The shutdown was initiated at `17:17:45` when considering the different timezone configurations.


Nevertheless, we can have an indicator, that the previous boot had an unclean shutdown by seeing if there was
- A recovery started during latest boot
- A filesystem check initiated during latest boot.

```sh
$ journalctl -b 0 |  grep -iE 'recover|fsck'
Apr 04 17:17:58 localhost systemd-fsck[651]: /usr/sbin/fsck.xfs: XFS file system.
Apr 04 17:17:58 localhost kernel: XFS (vda4): Starting recovery (logdev: internal) <===== HERE
Apr 04 17:17:58 localhost kernel: XFS (vda4): Ending recovery (logdev: internal) <===== HERE
Apr 04 17:17:59 compute-0 systemd[1]: Created slice Slice /system/systemd-fsck.
Apr 04 17:17:59 compute-0 systemd[1]: systemd-fsck-root.service: Deactivated successfully.
Apr 04 17:18:00 compute-0 systemd-fsck[819]: boot: recovering journal <===== HERE
Apr 04 17:18:00 compute-0 systemd-fsck[819]: boot: clean, 365/98304 files, 150426/393216 blocks
```
We see that the system did not go through a normal shutdown sequence due to the presence of `XFS` recovery and `fsck` journal recovery which confirms that filesystems were not cleanly unmounted. See the `unclean unmounts` section in the [RHEL 9 docs for XFS](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/checking-and-repairing-a-file-system__managing-file-systems#error-handling-mechanisms-in-xfs_checking-and-repairing-a-file-system).

## Case 6: A kernel panic occured

A kernel panic is a safety measure taken by the kernel of an operating system when an internal fatal error is detected, such as hardware failures and memory corruption. First we need to clarify, if a system would reboot in such a case. The responsible setting is `kernel.panic`, where `0` means that the system is not rebooted. Any other value indicates the number of seconds which are waited before the reboot is triggered, see further [kernel docs](https://docs.kernel.org/admin-guide/sysctl/kernel.html#panic).

```sh
$ sysctl kernel.panic
kernel.panic = 10
```

If a kernel panic would have happened, we would usually find a corresponding file in `/sys/fs/pstore/`
```sh
$ ls /sys/fs/pstore/
sh-5.1# 
```

As the setup here is a bit more complicated for OpenShift, I will leave it without an example. For further readings, please follow the [kernel docs](https://docs.kernel.org/admin-guide/pstore-blk.html).

## Conclusion

Linux offers many powerful utilities to understand what has happened on your system. This includes user initiated actions, hypervisor / cloud provider activity and even hardware failures. If you have other indicators which you are usually verifying, please let me know. 