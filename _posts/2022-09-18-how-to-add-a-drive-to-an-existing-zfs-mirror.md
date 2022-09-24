---
title: How to add a new drive to an existing ZFS mirror
date: 2022-09-18 14:00:00 -500
categories: [homelab, zfs]
tags: [homelab,zfs]
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

This article focuses on step 3.

## How to add a new drive to an existing ZFS mirror

In short, once the new drive is available on the machine, run the following:

```bash
zpool attach [poolname] [original drive to be mirrored] [new drive]
```

```bash
# Example:
zpool attach boot-pool /dev/sdj /dev/sdm
```

Thats pretty much it! You can track the resilvering process by running `zpool status boot-pool` , just replace
boot-pool with the name of your pool.
