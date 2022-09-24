---
title: How to create a raid 01 pool using ZFS
date: 2022-09-24 16:00:00 -500
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

While doing step 1, I ran into an interesting problem. The SUN F40 is a peculiar drive in that it is essentially 4
100Gb SAS SSDs strapped onto a raid controller, configured in IT mode. This means each of the SSDs present as
individual drives to the operating system. In my case though, I want to use these as two logical redundant drives
mirrored together, so that if the raid controller itself fails, I still have all of my data.[^1]

This would typically be referred to as a RAID 01 configuration, where the drives on each adapter are striped together,
and then those new striped volumes are mirrored. ZSF does not explicitly support a RAID 01 configuration, and in this
article, we will dive into why, and how to get around it.

## ZFS pool layout

In ZFS, pools are made up of vdevs, and vdevs are made up of hard drives. Drives cannot be shared between vdevs, vdevs
are responsible for their redundancy, and data is automatically distributed across vdevs inside of a pool. Redundancy
options for a ZFS pool are mirror, raidz1, raidz2, and raidz3. Mirror means all of the drives have all of the
data in the vdev, and there is no data loss as long as one drive is still alive in the vdev. The raidz(x) options
reflect the number of drives that are dedicated to redundancy. So, for raidz1, one drive can be lost without losing any
data. For raidz2, 2 drives can be lost, and for raidz3, 3 drives can be lost.

So, because vdevs are responsible for their redundancy, if I were to create two mirror vdevs and put all of the drives
from one adapter in one mirror vdev, and then all of the drives from the other adapter in another mirror vdev, I would
end up with two 100Gb sized vdevs, with redundancy for only drive failures on each adapter. If one adapter goes
missing, all of the data would be lost.

So, ZFS is always redundancy first (inside a vdev) and stripe second (across vdevs in a pool), where we want the
opposite. How do we accomplish this with the tools that we have?

## The Solution

We can still leverage ZFS to solve this problem with a little creativity. The solution is to create 4 mirrored vdevs in
our pool, each with one drive from each adapter. Using the command line, this would look something like the following:

```bash
zpool create poolname mirror ad1d1 ad2d1 mirror ad1d2 ad2d2 ...
```

Where "ad" stands for adapter, and "d" stands for drive. Visually, this gives us the following layout:

![zfs pool layout](/assets/img/2022-09-24-how-to-expand-a-zfs-pool-partitions/zfs-pool-layout.png)
_The ZFS Pool Layout_

That's it! There is no real special sauce here, just using the system to get what we want. It is also worth noting that
this is technically more redundant than a RAID 01. If one drive were to die on both adapters with a RAID 01, all data
would be lost. With this setup, we could lose multiple dives on both adapters, as long as they are not part of the
same vdev.

## Mermaid code for diagram
```
C4Context
  Boundary(b0, "ZFS pool", "") {

    Boundary(b1, "Adapter 1", "") {
        System(ad1d1, "Drive 1", "")
        System(ad1d2, "Drive 2", "")
        System(ad1d3, "Drive 3", "")
        System(ad1d4, "Drive 4", "")
    }

    Boundary(b2, "Adapter 2", "") {
        System(ad2d1, "Drive 1", "")
        System(ad2d2, "Drive 2", "")
        System(ad2d3, "Drive 3", "")
        System(ad2d4, "Drive 4", "")
    }
  }

BiRel(ad1d1, ad2d1, "vdev 1 (mirror)")
UpdateRelStyle(ad1d1, ad2d1, $textColor="white", $lineColor="red", $offsetX="-25")
BiRel(ad1d2, ad2d2, "vdev 2 (mirror)")
UpdateRelStyle(ad1d2, ad2d2, $textColor="white", $lineColor="red", $offsetX="-25")
BiRel(ad1d3, ad2d3, "vdev 3 (mirror)")
UpdateRelStyle(ad1d3, ad2d3, $textColor="white", $lineColor="red", $offsetX="-25")
BiRel(ad1d4, ad2d4, "vdev 4 (mirror)")
UpdateRelStyle(ad1d4, ad2d4, $textColor="white", $lineColor="red", $offsetX="-25")

UpdateElementStyle(b0, $fontColor="white", $borderColor="white")
UpdateElementStyle(b1, $fontColor="white", $borderColor="white")
UpdateElementStyle(b2, $fontColor="white", $borderColor="white")

UpdateLayoutConfig($c4ShapeInRow="1", $c4BoundaryInRow="2")
```

## Footnotes

[^1]: While I could flash the drives into IR mode, have them present as one logical disk, and then mirror those disks, the [flashing process](https://kasilag.me/warpdrive/) is quite involved, and I would also lose SMART data on the SSDs themselves. I am probably losing some performance on the table by not doing this, but considering the trade-offs here I decided to let ZFS manage the drives, rather than flash the raid controllers.
