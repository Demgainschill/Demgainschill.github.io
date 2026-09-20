---
title: 'LVM on RAID'
description: 'Build the array first. Let LVM slice what already survives a disk death.'
pubDate: 'Sep 20 2026'
heroImage: '../../assets/lvm-on-raid2.jpg'
---

The stack is not a taste. It is a failure domain.

RAID owns the spindles. LVM owns the slices. Put them in that order and a dead disk is one story: the array degrades, you replace a member, you wait for resync. Invert them and the same death becomes two stories at once — an LVM event and a RAID event — and you will spend the night deciding which tool is telling the truth.

This is the layout:

```text
disks → mdadm (or a hardware RAID device) → one block device → PV → VG → LVs → filesystems
```

Four members underneath. One volume group on top. The picture on this page is that sentence, made physical.

<div class="split">
<div class="pane">

### Left: the array

mdadm (or the controller) turns member disks into a single device that already knows how to lose one of them. RAID1 and RAID10 are the sane defaults for a machine you have to boot and think on. RAID5/6 wait until you have measured rebuild time on *your* disks, not a datasheet.

The array is the only place that should speak in disk serials.

```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb
mdadm --detail --scan >> /etc/mdadm.conf
update-initramfs -u
```

</div>
<div class="pane">

### Right: the volume group

Once `/dev/md0` exists, LVM should see one PV. Not four. One.

From that PV you cut root, var, home, a scratch LV, a thin pool later — none of those decisions belong in the RAID layer. Growing an LV is `lvextend`. Moving a filesystem to a new array later is `pvmove`. Those tools stay boring only if they sit on a device that is already redundant.

```bash
pvcreate /dev/md0
vgcreate vg0 /dev/md0
lvcreate -n root -L 40G vg0
lvcreate -n var  -L 20G vg0
mkfs.ext4 /dev/vg0/root
```

</div>
</div>

## Why this order

A RAID device is a lie that you want. It pretends four disks are one. LVM is also a lie: it pretends one device is many. Lies compose cleanly in one direction. The lower lie must be the one that survives hardware. The upper lie must be the one that survives *you* changing your mind about sizes.

If LVM sits on raw disks and RAID sits on the LVs, then:

- A member failure hits a physical volume that LVM still thinks is healthy until I/O errors climb the stack.
- `/boot` and the initramfs have to assemble LVM *before* they can assemble the array that the LV was supposed to be on. That path exists. It is not a path you want to debug at 2 a.m.
- Snapshots, thin pools, and `pvmove` all assume the PV is a stable block device. An array under them is stable. An array *made of them* is not.

Hardware RAID is the same rule with a different badge. The controller presents `/dev/sda`. That `sda` is already the array. You still run `pvcreate` on it. You do not build mdadm on top of the card, and you do not hand LVM the individual member disks the BIOS can still see.

## What belongs where

| Question | Layer |
|---|---|
| A disk died. What do I replace? | RAID |
| Root is tight. What do I grow? | LVM |
| I need a new scratch filesystem. | LVM |
| Rebuild / bitmap / write-intent. | RAID |
| Snapshot before an upgrade. | LVM |
| Two machines should see the same LUNs. | Multipath, then LVM — still not RAID-on-LV |

Keep `/boot` simple. On Debian and on RHEL, a small RAID1 of `/boot` (or an EFI partition on each disk) plus LVM on the remaining array is the layout that still boots when one drive is on the desk.

## The other RAID people mean

`lvcreate --type raid1` is not “RAID on LVM” in the bad sense. It is LVM asking device-mapper to do RAID *for that logical volume*. Use it when two LVs on the same disks need different redundancy — a RAID1 journal next to a RAID10 data LV. The members are still physical volumes. The filesystem still sits on a redundant LV. The stack did not flip.

What you should not do:

```text
disks → PV → VG → LV → mdadm on those LVs → filesystem
```

That is RAID on LVM. It looks clever in a notebook. It is how you get two spare-disk stories and no clean `mdadm --detail`.

## A stack you can rebuild from memory

```bash
# members already wiped and partitioned as type Linux RAID
mdadm --create /dev/md0 --level=10 --raid-devices=4 \
  /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1

pvcreate /dev/md0
vgcreate vg0 /dev/md0

lvcreate -n root -L 40G vg0
lvcreate -n var  -L 30G vg0
lvcreate -n home -l 100%FREE vg0

mkfs.ext4 -L root /dev/vg0/root
mkfs.ext4 -L var  /dev/vg0/var
mkfs.ext4 -L home /dev/vg0/home
```

Then fstab points at `/dev/vg0/*` or at UUIDs. It never points at `/dev/sdX`. The day a cable moves, the RAID name stays. The VG stays. The only file that mentions serials is `mdadm.conf`.

## The test

Pull a disk on a machine that can afford it. If the first command you reach for is `mdadm --detail`, the stack is right. If the first command you reach for is `pvs` because you cannot tell whether the PV vanished or the array did, the stack is upside down.

Build the array first. Let LVM slice what already survives a disk death.
