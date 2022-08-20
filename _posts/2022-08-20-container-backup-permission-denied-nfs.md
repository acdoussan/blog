---
title: 'How to fix "Cannot open: Permission denied" when trying to backup an unprivileged LXC container over NFS in Proxmox'
date: 2022-08-20 15:55:00 -500
categories: [homelab, proxmox]
tags: [homelab,proxmox,lxc,container,containers]
image:
  path: /assets/img/2022-08-20-container-backup-permission-denied-nfs/proxmox-error.png
  width: 1590
  height: 998
  alt: Screenshot of the error when trying to take a backup
---

# The Problem

When trying to take a full stop backup of my LXC containers in Proxmox, I get the following error output:

```
INFO: starting new backup job: vzdump 107 --mode stop --compress zstd --remove 0 --node hp2 --storage truenas-nfs-hdd --notes-template '{{guestname}}'
INFO: Starting Backup of VM 107 (lxc)
INFO: Backup started at 2022-08-20 15:12:12
INFO: status = running
INFO: backup mode: stop
INFO: ionice priority: 7
INFO: CT Name: pihole-secondary
INFO: including mount point rootfs ('/') in backup
INFO: stopping virtual guest
INFO: creating vzdump archive '/mnt/proxmox-nfs-hdd/dump/vzdump-lxc-107-2022_08_20-15_12_12.tar.zst'
INFO: tar: /mnt/proxmox-nfs-hdd/dump/vzdump-lxc-107-2022_08_20-15_12_12.tmp: Cannot open: Permission denied
INFO: tar: Error is not recoverable: exiting now
INFO: restarting vm
INFO: guest is online again after 5 seconds
ERROR: Backup of VM 107 failed - command 'set -o pipefail && lxc-usernsexec -m u:0:100000:65536 -m g:0:100000:65536 -- tar cpf - --totals --one-file-system -p --sparse --numeric-owner --acls --xattrs '--xattrs-include=user.*' '--xattrs-include=security.capability' '--warning=no-file-ignored' '--warning=no-xattr-write' --one-file-system '--warning=no-file-ignored' '--directory=/mnt/proxmox-nfs-hdd/dump/vzdump-lxc-107-2022_08_20-15_12_12.tmp' ./etc/vzdump/pct.conf ./etc/vzdump/pct.fw '--directory=/mnt/vzsnap0' --no-anchored '--exclude=lost+found' --anchored '--exclude=./tmp/?*' '--exclude=./var/tmp/?*' '--exclude=./var/run/?*.pid' ./ | zstd --rsyncable '--threads=1' >/mnt/proxmox-nfs-hdd/dump/vzdump-lxc-107-2022_08_20-15_12_12.tar.dat' failed: exit code 2
INFO: Failed at 2022-08-20 15:12:17
INFO: Backup job finished with errors
TASK ERROR: job errors
```

# Research

[Some posts](https://forum.proxmox.com/threads/lxc-unprivileged-backup-task-failing.48565/post-227443)
recommend setting a local temp directory in `/etc/vzdump.conf` for each node. While this would work, my nodes do not
have a lot of local storage, and there is the potential for them to run out of space when taking backups. Additionally,
I would rather not have the write cycles on the boot drives if possible.

There was [another post](https://forum.proxmox.com/threads/lxc-unprivileged-backup-task-failing.48565/post-465750)
that recommended verifying that the folders are actually writeable, and adjusting the permissions by running
`chmod 777` if they are not. There were no issues with the permissions set on my folders, so this is not my
issue, but it may be worth checking.

# The Solution

[This post](https://forum.proxmox.com/threads/tmp-cannot-open-permission-denied.87730/post-441028) mentions mapping
all users to root as part of the NFS configuration. Because LXC containers use
[linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces), the user in an unprivileged container will not be
root, and therefore you have to map all users. On the configuration on my NFS share in TrueNAS, I moved my NFS user
from the "maproot" section to the "mapall" section, and sure enough, backups work!

![TrueNAS Configuration](/assets/img/2022-08-20-container-backup-permission-denied-nfs/truenas.png){: width="1590" height="998" }
_Screenshot of configurating the NFS share in TrueNAS_
