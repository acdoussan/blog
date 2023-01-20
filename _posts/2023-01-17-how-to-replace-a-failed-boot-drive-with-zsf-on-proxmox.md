---
title: How to Replace a Failed Boot Drive in a ZFS Mirror on Proxmox
date: 2023-01-17 17:00:00 -500
categories: [homelab, proxmox]
tags: [homelab,zfs,proxmox]
---

## The Problem

I recently noticed one of my proxmox nodes that would usually take about 30 seconds to reboot would take about 15
minutes to reboot. After a bit of investigating, it turns out that one of my boot drives had failed, and it was
stalling the boot of the machine on the splash screen, presumably because the bios was still trying to communicate
with it.

As it turns out, replacing a failed boot drive is not quite as easy as replacing a failed drive in zfs, so I figured
I might as well document it, if for no one other than future me. There is some
[proxmox documentation](https://pve.proxmox.com/wiki/ZFS_on_Linux#sysadmin_zfs_change_failed_dev) but it is not the
best.

## Steps to replace the drive

First, we will start with `zpool status`. You should see something like the following:

```
root@hp1:~# zpool status
  pool: rpool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
  scan: scrub repaired 0B in 12:17:31 with 0 errors on Sun Dec 11 12:41:32 2022
config:

	NAME                                                    STATE     READ WRITE CKSUM
	rpool                                                   DEGRADED     0     0     0
	  mirror-0                                              DEGRADED     0     0     0
	    14568071368226837248                                UNAVAIL      0     0     0  was /dev/nvme0n1p3
	    ata-PNY_CS900_240GB_SSD_PNY22092203030100C65-part3  ONLINE       0     0     0
```

It would be a good idea to try and get the drive alive again before proceeding (reseat the drive, inspect the area for
dust, etc) but we're going to assume the drive is good and dead and continue. The first step is to physically replace
the drive with a suitable (ideally identical) replacement. The drive must at least be the same size or bigger.

Once the drive has been replaced physically, we can move on to the next step. We need to copy the partitions from the
good drive to the new one so that the machine can boot from it.

### Copy the partitions to the new drive from the good drive

> Please read these steps carefully, you have a chance of losing your data if you do it wrong.
{: .prompt-warning }


1. First, run `lsblk` and look for the new drive (likely has the same name as the previous drive, in our case,
`nvme0n1`).

2. Copy the partitions from the good drive to the new drive using `sgdisk --replicate=/dev/TARGET /dev/SOURCE`. BE CAREFUL HERE, if you get the command backward, you will lose all of your data on the good drive. In our case, I ran `sgdisk --replicate=/dev/nvme0n1 /dev/sda`

3. We need to randomize the GUIDs to make sure weird things don't happen with zfs. We can do this with `sgdisk --randomize-guids <NEW DRIVE>`. In our case, I ran `sgdisk --randomize-guids /dev/nvme0n1`.

### Add the drive to the ZFS mirror

Now that the drive has been formatted correctly, we can add it to the mirror with `zpool replace`. `zpool replace` takes
in the pool, the drive to replace, and the new drive, so our  example `zpool replace rpool /dev/nvme0n1p3
/dev/nvme0n1p3`. Because our device name did not change, we could have also used the shorthand `zpool replace rpool
/dev/nvme0n1p3` for this. Make sure to use the 3rd partition for this, as that is where the data is stored.

In my case, I ended up using the by-id listing of the drive to keep things consistent. I'm not sure if this makes any
difference or what the tradeoffs (if any) are here. I ended up running `zpool replace rpool /dev/nvme0n1p3
/dev/disk/by-id/nvme-TEAM_TM8FP6256G_TPBF2207080020101623-part3`.

### Give ZFS time to resilver

If you run zpool status again, you should see something like the following:

```
zpool status
  pool: rpool
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Mon Jan 16 14:37:35 2023
	5.42G scanned at 1.35G/s, 605M issued at 151M/s, 5.42G total
	631M resilvered, 10.90% done, 00:00:32 to go
config:

	NAME                                                    STATE     READ WRITE CKSUM
	rpool                                                   ONLINE       0     0     0
	  mirror-0                                              ONLINE       0     0     0
	    ata-PNY_CS900_240GB_SSD_PNY22092203030100C65-part3  ONLINE       0     0     0
	    nvme-TEAM_TM8FP6256G_TPBF2207080020101623-part3     ONLINE       0     0     0  (resilvering)
```

Let the resilver finish before continuing. When it is done, you should see something like the following:

```
zpool status
  pool: rpool
 state: ONLINE
  scan: resilvered 5.61G in 00:00:30 with 0 errors on Mon Jan 16 14:38:05 2023
config:

	NAME                                                    STATE     READ WRITE CKSUM
	rpool                                                   ONLINE       0     0     0
	  mirror-0                                              ONLINE       0     0     0
	    ata-PNY_CS900_240GB_SSD_PNY22092203030100C65-part3  ONLINE       0     0     0
	    nvme-TEAM_TM8FP6256G_TPBF2207080020101623-part3     ONLINE       0     0     0
```

### Configure proxmox-boot-tool

> This is assuming you are using proxmox 6.3 or greater, and if you started with proxmox 6.2 or newer, you have migrated off of grub. If you are still using grub, `grub-install <new disk>` should be what you want, but I have not tried or tested this, and you should look at the proxmox documentation before continuing.
{: .prompt-warning }

In short, we need to run two commands on the second partition of the new drive.

1. `proxmox-boot-tool format /dev/nvme0n1p2`
2. `proxmox-boot-tool init /dev/nvme0n1p2`

This properly sets up the second partition and the drive should now be bootable. You can check your work with
`proxmox-boot-tool status`

```
root@hp1:~# proxmox-boot-tool status
Re-executing '/usr/sbin/proxmox-boot-tool' in new private mount namespace..
System currently booted with uefi
WARN: /dev/disk/by-uuid/CE07-1118 does not exist - clean '/etc/kernel/proxmox-boot-uuids'! - skipping
6D31-87F4 is configured with: uefi (versions: 5.13.19-6-pve, 5.15.39-3-pve, 5.15.83-1-pve)
CE07-78FC is configured with: uefi (versions: 5.13.19-6-pve, 5.15.39-3-pve, 5.15.83-1-pve)

```

The last step is to clean up the dangling dead drive uuid. You can do this by running `proxmox-boot-tool clean`.

```
root@hp1:~# proxmox-boot-tool clean
Checking whether ESP '6D31-87F4' exists.. Found!
Checking whether ESP 'CE07-1118' exists.. Not found!
Checking whether ESP 'CE07-78FC' exists.. Found!
Sorting and removing duplicate ESPs..
```

After that, check one more time with `proxmox-boot-tool status`.

```
root@hp1:~# proxmox-boot-tool status
Re-executing '/usr/sbin/proxmox-boot-tool' in new private mount namespace..
System currently booted with uefi
6D31-87F4 is configured with: uefi (versions: 5.13.19-6-pve, 5.15.39-3-pve, 5.15.83-1-pve)
CE07-78FC is configured with: uefi (versions: 5.13.19-6-pve, 5.15.39-3-pve, 5.15.83-1-pve)
```

And that is it, your new drive should be good to go! It would be wise to unplug your known good drive and make sure your
system can boot from the new drive, but everything should be configured at this point. I hope your new drive lasts
forever!
