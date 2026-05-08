# Nordix ZGuide - Zvol as L2ARC 

## **I will only recommend using L2ARC for HHD zpool's**

---

 > in my computer I have 4 nvme gen4 in zfs stripe for my system
 > I also have a zpool consisting of 4 hard drives with two SATA SSD drives as a special vdev, it is created with a redundancy part for which I will also create an L2ARC.
</br>
 > I believe that running my storage pool tank with primarycache=all is unnecessary 
 > and you have more important data to cache in ARC than my media pool has so i run primarycache=metdata on my storage pools 
</br>
 > instead I will create a zvol on my system zpool
 > the one I have 4 nvme in stripe which I will use as L2ARC for my HHD pool

---

## Zvol creation and options

 - volblocksize: virtual sector size, you can try to set this more or less but but probably 16k is a robust choice 
 
 - compression: This is something you can experiment with, if you want to do this I recommend lz4 or zstd-fast-1000
 
 - sync: We turn off sync as this is only a cache and does not need any form of check.
 
 - redundant_metadata: how many backup copies of metadata, we still have good redundancy with most vs all. in this case you might be able to go lower, I put most on everything I run so I do this too
 
 - logbias: Set it to either throughput or latency depending on whether you want to prioritize read or write
 
 - primarycache: here you determine what to cache in ARC, you can choose between - all, metadata, none, this with zvol as L2ARC is a bit experimental, but I think it's good to run on metadata

 - secondarycache: is whether we should cache in L2ARC, since this zvol will act as L2ARC we set this option to none, otherwise you have the same choice as with primarycache
 
 - checksum: same as sync, this is only a cache there fore we set this options to checksum=off, otherwise in general checksum=fletcher4 is a solid choice

**The last thing is which zpool I want to create this zvol on and in this case it is zpool nordix, I choose to name my zvol zvol-l2arc, so then it becomes: nordix/zvol-l2arc-tank and nordix/zvol-l2arc-tankz1**

```Fish
sudo zfs create -V 100G \
   -o volblocksize=16K \
   -o compression=off \
   -o sync=disabled \
   -o redundant_metadata=most \
   -o logbias=throughput \
   -o primarycache=metadata \
   -o secondarycache=none \
   -o checksum=off \
   nordix/zvol-l2arc-tank
```
```Fish
sudo zfs create -V 50G \
   -o volblocksize=16K \
   -o compression=off \
   -o sync=disabled \
   -o redundant_metadata=most \
   -o logbias=throughput \
   -o primarycache=metadata \
   -o secondarycache=none \
   -o checksum=off \
   nordix/zvol-l2arc-tankz1
```

**Locate which zpool we should use**

```Fish
zpool list
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
nordix     3.62T   995G  2.65T        -         -     2%    26%  1.00x    ONLINE  -
tank       19.4T  4.24T  15.1T        -         -     0%    21%  1.00x    ONLINE  -
tank-z1    5.06T   139G  4.93T        -         -     0%     2%  1.00x    ONLINE  -
```

I will create zvol - L2ARC for my zpool tank, but I will also create one for my redundancy pool tank-z1 to hopefully get faster writes to it. This may need to be tweaked and I already have a zfs.conf file suitable for desktops, but I will go into more depth on what can be done to make L2ARC more efficient as well.

---

## Add cache to zpool 

**now we will use these zvols as cache for my HHD zpools**

zvol-l2arc-tank will be cache for tank
zvol-l2arc-tankz1 will be cache for tank-z1

 > You need the full path to the zvol 
 > You find the zvol's in /dev/zvol/*
 
Full path for zvol-l2arc-tank is:
 - /dev/zvol/nordix/zvol-l2arc-tank

Full path for zvol-l2arc-tankz1 is:
 - /dev/zvol/nordix/zvol-l2arc-tankz1

**Add zvol-l2arc-tank as cache to tank:**
```Fish
sudo zpool add tank cache /dev/zvol/nordix/zvol-l2arc-tank
```

**Add zvol-l2arc-tankz1 as cache to tank-z1:**
```Fish
sudo zpool add tank-z1 cache /dev/zvol/nordix/zvol-l2arc-tankz1
```

**Done!**

We can now check if we have succeeded.
```Fish
zpool status tank
  pool: tank
 state: ONLINE
config:

	NAME                                            STATE     READ WRITE CKSUM
	tank                                            ONLINE       0     0     0
	  ata-HUH728080ALN600_2EHXRH1X-part1            ONLINE       0     0     0
	  ata-HUH728080ALN600_VJG1U0SX-part1            ONLINE       0     0     0
	  ata-HUH728080ALN600_VJG1PJBX-part1            ONLINE       0     0     0
	  ata-HUH728080ALN600_2EGRERKX-part1            ONLINE       0     0     0
	special	
	  mirror-4                                      ONLINE       0     0     0
	    ata-INTEL_SSDSC2CW120A3_CVCV430601BD120BGN  ONLINE       0     0     0
	    ata-KINGSTON_SA400S37120G_50026B767B0067D9  ONLINE       0     0     0
	cache
	  zvol/nordix/zvol-l2arc-tank                   ONLINE       0     0     0  block size: 4096B configured, 16384B native

errors: No known data errors
```

```Fish
zpool status tank-z1
  pool: tank-z1
 state: ONLINE
config:

	NAME                                    STATE     READ WRITE CKSUM
	tank-z1                                 ONLINE       0     0     0
	  raidz1-0                              ONLINE       0     0     0
	    ata-HUH728080ALN600_2EHXRH1X-part2  ONLINE       0     0     0
	    ata-HUH728080ALN600_VJG1U0SX-part2  ONLINE       0     0     0
	    ata-HUH728080ALN600_VJG1PJBX-part2  ONLINE       0     0     0
	    ata-HUH728080ALN600_2EGRERKX-part2  ONLINE       0     0     0
	cache
	  zvol/nordix/zvol-l2arc-tankz1         ONLINE       0     0     0  block size: 4096B configured, 16384B native

errors: No known data errors
```

**now we can clearly see that my zpools have zvol as cache. 
it is worth noting that unlike special vdev, you can remove such a cache without damaging your zpool**



