---
title: How to resize the partitions of a drive that is already part of a ZFS pool
date: 2022-09-24 13:00:00 -500
categories: [homelab, zfs]
tags: [homelab,zfs,partitions]
---

## The Problem

I recently got a pair of Sun F40's to use as boot drives for proxmox on my Dell r720. Previously, I booted proxmox off
of a pair of internal SD cards. I passed all of the drives through to a VM, which would then boot truenas off of
connected SSDs by passing through the HBA directly to the VM using IOMMU. I then stored my VMs on SSDs managed by
truenas by connecting to an NFS share hosted on truenas.

This is obviously pretty complicated, so I bought the Sun F40's with the intent of migrating all of my VMs to use local
storage, while also getting rid of the SD cards as they are not the most reliable storage, and also have limited
capacity.

So, the general plan was:

1. Install the new drives, reinstall proxmox, and restore the VMs from backups
2. Migrate all of the VMs that already had virtual drives to the new local storage
3. Create a local storage drive for truenas and add it to the existing boot mirror
4. Delete the NFS share, remove the SSDs from the truenas boot pool, and reuse them for something else

While doing step 1, the SD cards were temporarily included in the mirror I was creating. This meant that the size of the
pool was limited to about 10Gb per drive when 100Gb was available. When this happened, I had two options:

1. Reinstall proxmox and set the size correctly
2. Fix it

Given I already had the server up and running before I noticed this mistake, I decided to try and fix it.

## How to resize the partitions and expand the pool

To expand the partitions, we are going to use `parted`. It should be available from the package manager on most
distros, but you will likely have to install it yourself.

First, we need to find the drives and partitions that need to be expanded. Sometimes ZFS will list the `/dev/sd[a,b,c,...]` ID in the output of `zpool status`, but in my case, I was seeing the following:

```
root@r720:~# zpool status rpool
  pool: rpool
 state: ONLINE
  scan: scrub repaired 0B in 00:00:17 with 0 errors on Sun Sep 11 00:24:18 2022
config:

        NAME                                     STATE     READ WRITE CKSUM
        rpool                                    ONLINE       0     0     0
          mirror-0                               ONLINE       0     0     0
            ata-3E128-TS2-550B01_5L004JX7-part3  ONLINE       0     0     0
            ata-3E128-TS2-550B01_FL006YLH-part3  ONLINE       0     0     0
          mirror-1                               ONLINE       0     0     0
            ata-3E128-TS2-550B01_5L004JYD-part3  ONLINE       0     0     0
            ata-3E128-TS2-550B01_FL006YE9-part3  ONLINE       0     0     0
          mirror-2                               ONLINE       0     0     0
            ata-3E128-TS2-550B01_5L004KD9-part3  ONLINE       0     0     0
            ata-3E128-TS2-550B01_FL006YXP-part3  ONLINE       0     0     0
          mirror-3                               ONLINE       0     0     0
            ata-3E128-TS2-550B01_5L004KL2-part3  ONLINE       0     0     0
            ata-3E128-TS2-550B01_FL006YEB-part3  ONLINE       0     0     0
```

These are the "by id" identifiers for these drives. To find the `/dev/sd[a,b,c,...]` that these identifiers
correlate to, you can run the following:

```bash
ls -l /dev/disk/by-id
```

```
Example Output:
lrwxrwxrwx 1 root root  9 Sep 24 14:07 ata-3E128-TS2-550B01_5L004JX7 -> ../../sdq
lrwxrwxrwx 1 root root 10 Sep 24 14:07 ata-3E128-TS2-550B01_5L004JX7-part1 -> ../../sdq1
lrwxrwxrwx 1 root root 10 Sep 24 14:07 ata-3E128-TS2-550B01_5L004JX7-part2 -> ../../sdq2
lrwxrwxrwx 1 root root 10 Sep 24 14:07 ata-3E128-TS2-550B01_5L004JX7-part3 -> ../../sdq3
```

Bonus note: According to
[this post](https://forums.FreeBSD.org/threads/zpool-confusion-where-is-my-partition.63555/post-367802),
if you add the following lines to `/boot/loader.conf` and reboot your machine `zpool status` will show the `sd[x]`
names instead. I have not tried this, do so at your own risk.

```
kern.geom.label.disk_ident.enable="0"           # Disable the auto-generated Disk IDs  for disks
kern.geom.label.gptid.enable="0"        # Disable the auto-generated GPT UUIDs for disks
kern.geom.label.ufsid.enable="0"        # Disable the auto-generated UFS UUIDs for filesystems
```

Once you have the proper drive names, one by one open them up using `parted` and do the following:

```
root@r720:~# parted /dev/sdq
GNU Parted 3.4
Using /dev/sdq
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: ATA 3E128-TS2-550B01 (scsi)
Disk /dev/sdq: 100GB
Sector size (logical/physical): 512B/8192B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      17.4kB  1049kB  1031kB                     bios_grub
 2      1049kB  538MB   537MB   fat32              boot, esp
 3      538MB   10GB    9.5GB   zfs

(parted) resizepart 3 100%
(parted) print
Model: ATA 3E128-TS2-550B01 (scsi)
Disk /dev/sdq: 100GB
Sector size (logical/physical): 512B/8192B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      17.4kB  1049kB  1031kB                     bios_grub
 2      1049kB  538MB   537MB   fat32              boot, esp
 3      538MB   100GB   99.5GB  zfs

(parted) q
```

In short, you want to open parted on the disk using `parted /dev/sd[x]`, then run `print` to find the partition number,
then run `resizepart <partition number> 100%` to make the partition fill the disk, then `print` again to make sure it
worked, then `q` to quit. Repeat this for every drive you need to expand.

Once you have resized all of the partitions, you need to let ZFS know it can expand the pool to fill this new space.
For each drive listed in the pool, run the following.

```bash
zpool online -e <pool name> <disk name from zpool status>
```

```bash
# Example:
zpool online -e rpool ata-3E128-TS2-550B01_5L004JX7-part3
```

Once you have done that you should be done! If after doing this your pool still does not fill the entire space, try
setting autoexpand on the pool.

```bash
zpool set autoexpand=on <pool name>
```

```bash
# Example:
zpool set autoexpand=on rpool
```
