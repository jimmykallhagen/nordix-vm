# ZFS, ARC, zvol and Virtual Machines
### A conceptual guide to understanding how they work together

---
[Windows11 ZFS Setup With Benchmarks](/win11-zfs-setup-benchmark.md)
---

## The Core Misconception

A common recommendation you will find online is to **disable primarycache on a zvol used for a VM**. The reasoning often goes: *"The VM is a single large file, and ARC might try to cache the whole thing, wasting all your RAM."*

This is a misconception, and understanding *why* it is wrong will give you a much deeper understanding of ZFS.

---

## How ARC Actually Works

ARC (Adaptive Replacement Cache) does **not** work with files. It works with **blocks**.

A 50GB VM image is not stored as a single block. It is split across thousands of individual blocks on disk. ARC does not cache "the VM file" - it caches the individual **blocks** that are actually being accessed. Blocks that are read frequently become "hot" and stay in ARC. Blocks that are never touched never enter ARC at all.

### The warehouse analogy

Think of your storage pool as a warehouse:

- The **physical shelves** represent `ashift` - the fundamental unit the physical drive understands
- The **boxes on the shelves** represent blocks - the unit ZFS works with
- The **items inside the boxes** represent the actual data

When ARC fetches data, it carries out a box at a time. It does not carry out the entire warehouse just because you asked for one item.

This is why disabling `primarycache` on a VM zvol is often counterproductive - you are throwing away the precise, selective block-level caching that is exactly what ARC is good at.

---

## What is a zvol?

A regular ZFS dataset is a **filesystem**. ZFS understands files, directories, metadata, and how to split files into blocks (`recordsize`).

A zvol is different. It is a **raw block device** - ZFS steps out of the filesystem role and instead presents a virtual drive. No files, no directories, no filesystem semantics. Just a long sequence of addressable blocks.

This means:

- ZFS does **not** know what is inside the zvol
- The guest OS (your VM) builds its own filesystem (e.g. NTFS) inside it
- ZFS just stores and retrieves raw blocks on behalf of the guest

The zvol acts as a translation layer - a contract between two parties that do not know about each other:

```
VM / Guest OS          ZFS / Physical disk
(unaware of ZFS)  ←→  volblocksize  ←→  (unaware of NTFS)
```

---

## volblocksize vs recordsize

This is where zvols differ from regular datasets, and why the terminology changes.

| Concept | Dataset | zvol |
|---|---|---|
| Parameter | `recordsize` | `volblocksize` |
| ZFS knows about files? | Yes | No |
| Meaning | Largest block for a file record | Smallest addressable block unit |
| Analogy | Box size for packing files | Sector size of a virtual drive |

`recordsize` is a **filesystem concept** - it relates to how files are packed into blocks.

`volblocksize` is a **block device concept** - it defines the minimum unit ZFS reads and writes for this virtual drive, exactly like a physical sector size.

---

## The Three-Layer Stack

When a VM writes data, it passes through three layers, each with its own block size:

```
[ Guest OS (NTFS)   ]  → 4K clusters
        ↓
[ ZFS zvol          ]  → volblocksize (e.g. 8K or 16K)
        ↓
[ Physical disk     ]  → ashift (e.g. 4K sectors, ashift=12)
```

Ideally, these three layers speak the same language. When there is a **size mismatch**, the layer that received a smaller write than its block size must:

1. Read the entire block into memory
2. Modify the relevant portion
3. Write the entire block back to disk

This is called **read-modify-write** or **write amplification**, and it wastes I/O at every layer where a mismatch exists.

---

## zvol vs virtio image file (qcow2)

Both are ways to give a VM a virtual drive, but they are fundamentally different:

### virtio image file (e.g. qcow2):
- Lives as a **file** inside the host filesystem (ext4, xfs, etc.)
- VM writes blocks -> host filesystem translates to its own blocks → physical disk
- Two filesystems stacked on top of each other
- The host filesystem has no idea it is handling disk I/O - it treats the image like any other file

### zvol:
- ZFS **exits the filesystem role** entirely for this storage
- VM writes blocks -> ZFS handles them as raw blocks -> physical disk
- No intermediate filesystem that misunderstands what is happening
- ZFS retains all its strengths (CoW, compression, checksums, ARC) but hides them from the guest

The zvol is a more **genuine** virtual drive - the guest gets real block-device semantics, while ZFS silently handles data integrity and caching underneath.

---

## Choosing the Right volblocksize

Since `volblocksize` cannot be changed after the zvol is created, this decision matters.

The goal is to match `volblocksize` to the I/O pattern of the **guest OS**. Windows with NTFS typically performs random 4K writes. Linux guests vary by workload.

### Effect of block size on ARC caching

| volblocksize | ARC behaviour | Best for |
|---|---|---|
| Small (4K–8K) | Precise - only hot data is cached | Limited RAM, random I/O workloads |
| Medium (16K–32K) | Balanced | General purpose VMs |
| Large (64K–128K) | Coarse - neighbours of hot data also cached | Abundant RAM, sequential workloads |

### RAM budget matters

With **limited RAM** (e.g. 16GB shared between host, ARC and VM):
- Every ARC slot is valuable
- Smaller `volblocksize` means ARC caches only the precise blocks that are actually hot
- Large blocks would waste ARC space on cold data sitting next to hot data

With **abundant RAM** (e.g. 64GB+):
- You can afford some spatial "sloppiness"
- Larger blocks reduce I/O overhead and metadata pressure
- The cost of occasionally caching neighbouring cold data is negligible

### Practical recommendation

**16K** is a solid general-purpose choice for Windows VMs:
- Close enough to NTFS 4K clusters to avoid severe write amplification
- Not so small that ZFS metadata overhead becomes a problem
- Works well with ARC on typical RAM budgets

---

## Dedicated Storage for VMs

Running VM zvols on the **same pool as your host system** means the host and the VM compete for the same physical I/O queue. Every time the host reads a config file or writes a log, it shares bandwidth with the VM's disk activity. Under load this creates unpredictable latency for both.

A **dedicated pool or drive** for VM storage gives each workload its own I/O lane — the VM gets consistent, predictable access to its zvol without the host interfering, and vice versa. Even a secondary NVMe or SSD dedicated to VM storage makes a noticeable difference in VM responsiveness under heavy concurrent host activity.

---

## Compression, ARC and the CPU Trade-off

ZFS compression is off by default in vanilla OpenZFS, but enabling it is almost always the right choice. The key insight is that compression does not just save disk space — it directly improves performance, because a compressed block is smaller and therefore faster to read from disk and faster to store in ARC.

### Compressed ARC — multiplying your effective RAM

When `zfs_compressed_arc_enabled=1` is set, the ARC stores blocks in their **compressed form**. Data is only decompressed when an application actually reads it, not when it enters the cache.

```
Without compressed ARC:   disk -> decompress → ARC (uncompressed)
With compressed ARC:      disk -> ARC (compressed) → decompress on use
```

This means the ARC can hold **more data for the same amount of RAM**:

| Algorithm | Typical ratio | 40 GB ARC -> Effective capacity |
|---|---|---|
| lz4 | ~2.0x | ~80 GB |
| zstd | ~2.5x | ~100 GB |
| zstd (high) | ~3.0x | ~120 GB |

### Choosing a compression algorithm

The right choice depends on your CPU strength relative to your RAM budget:

| Algorithm | CPU cost | Compression | Best when |
|---|---|---|---|
| lz4 | Very low | Moderate | Weak CPU, or RAM is plentiful |
| zstd-3 | Low–medium | Good | General purpose, balanced systems |
| zstd-5 / zstd-7 | Medium–high | Very good | Strong CPU, limited RAM |
| zstd-19 | Extremely high | Maximum | Almost never worth it |

If you have a **powerful CPU but limited RAM**, a higher zstd level can meaningfully expand your effective ARC capacity - the CPU pays a small decompression cost at each cache hit, but you fit dramatically more useful data into the available RAM.

If you have **abundant RAM and a modest CPU**, lz4 is the better trade - fast decompression keeps latency low, and you have enough RAM that the extra ratio improvement from zstd is less critical.

---

## Nordix ZFS Configuration

If you are running [Nordix](https://github.com/nordix), the ZFS tuning described in this document is already handled for you - and then some.

Nordix ships a purpose-built `/etc/modprobe.d/zfs.conf` tuned for high-performance desktop systems on NVMe/SSD storage. Configurations are available for multiple RAM tiers:

**8 GB · 16 GB · 32 GB · 64 GB · 128 GB**

Each profile scales `arc_max`, `arc_min`, dirty data limits, and I/O queue depths appropriately for that RAM budget.

### What Nordix configures for you (64 GB profile as example)

**Compressed ARC** is enabled:
```
options zfs zfs_compressed_arc_enabled=1
```
With the 64 GB profile targeting ~37 GB `arc_max` and a typical ~2.5x compression ratio, the effective ARC capacity is roughly **100–120 GB** on a machine with 64 GB of physical RAM.

**ARC bounds** leave comfortable headroom for applications and VM RAM while maximising cache:
```
options zfs zfs_arc_max=40055800320   # ~37 GB
options zfs zfs_arc_min=15000000000   # ~14 GB
```
The minimum prevents cache thrashing - even under heavy application load, at least 14 GB of hot data stays in ARC without being evicted.

**I/O parallelism** is tuned for NVMe's multi-queue architecture:
```
options zfs zfs_vdev_async_read_max_active=32
options zfs zfs_vdev_sync_read_max_active=32
```
The OpenZFS defaults were designed for spinning disks. On NVMe, high queue depth is an advantage — these values exploit the device's ability to handle many concurrent operations simultaneously.

**Write buffering** is generous:
```
options zfs zfs_dirty_data_max_percent=40
options zfs zfs_txg_timeout=5
```
40% of RAM as dirty data buffer lets ZFS coalesce many small writes into large sequential batches before committing, reducing write amplification and SSD wear. The 5-second TXG timeout keeps commits frequent enough that data loss on power failure is bounded to a few seconds.

**LBA weighting is disabled** for SSD/NVMe:
```
options zfs metaslab_lba_weighting_enabled=0
```
This is a legacy optimisation for spinning disks that actively hurts NVMe by concentrating writes at the beginning of the device. Disabling it gives more even wear distribution across the full drive.

### What this means for your VM setup on Nordix

If you are running Nordix and following the VM guidance in this document, you do not need to manually configure compressed ARC or tune ARC bounds - it is already done and optimised for your RAM tier. Your job is simply to:

1. Create your zvol with the appropriate `volblocksize` for your guest OS
2. Set `compression=zstd` on the zvol (or `lz4` if CPU headroom is a concern)
3. Leave `primarycache=all` (the default) - do not disable it
4. Place VM storage on a dedicated pool if possible

The Nordix ZFS config then handles everything underneath transparently.

### Scaling guide across RAM tiers

| System RAM | arc_max | arc_min | Effective ARC* |
|---|---|---|---|
| 16 GB | ~8 GB | ~2 GB | ~16–24 GB |
| 32 GB | ~20 GB | ~8 GB | ~50–60 GB |
| 64 GB | ~37 GB | ~14 GB | ~100–120 GB |
| 128 GB | ~80 GB | ~30 GB | ~200–240 GB |

*Effective capacity assumes compressed ARC with approximately 2.5x ratio. Actual ratio depends on your data and compression algorithm.

---

## Summary

| Concept | Key insight |
|---|---|
| ARC caches blocks, not files | A 50GB VM will never flood your RAM - only accessed blocks are cached |
| Do not blindly disable primarycache | You lose the precise block-level caching that benefits VMs the most |
| zvol is a raw block device | ZFS steps out of filesystem role - no recordsize, only volblocksize |
| volblocksize = virtual sector size | Match it to the guest OS I/O size to avoid write amplification |
| Three layers, three block sizes | ashift -> volblocksize -> guest cluster size should be harmonious |
| zvol vs image file | zvol gives the guest genuine block semantics; image files add a redundant filesystem layer |
| Dedicated pool for VMs | Eliminates I/O competition between host and guest workloads |
| Compressed ARC multiplies capacity | More data fits in the same RAM - CPU pays a small decompression cost per read |
| Algorithm choice depends on CPU/RAM ratio | More RAM -> lz4 is fine. Less RAM, stronger CPU → zstd-5/7 earns its keep |
| Nordix handles ZFS tuning for you | Compressed ARC, I/O depths, write buffers - all pre-configured per RAM tier |

---

## Quick Reference

```bash
# Create a zvol for a Windows VM (16K volblocksize recommended)
zfs create -V 50G -b 16K -o compression=zstd -o primarycache=all tank/vm-windows

# Create a zvol for a Linux VM (8K volblocksize, more precise caching)
zfs create -V 50G -b 8K -o compression=zstd -o primarycache=all tank/vm-linux

# Check volblocksize of an existing zvol
zfs get volblocksize tank/vm-windows

# Check compression ratio
zfs get compressratio tank/vm-windows

# Verify compressed ARC is active
cat /sys/module/zfs/parameters/zfs_compressed_arc_enabled

# Quick ARC hit rate (target: >90%)
awk '/^hits/{h=$3} /^misses/{m=$3} END{printf "%.1f%%\n",h/(h+m)*100}' \
  /proc/spl/kstat/zfs/arcstats

# Monitor I/O per vdev in real time
zpool iostat -v <zpool>

# Pool health check
zpool status -x
```

> **Note:** `volblocksize` cannot be changed after creation. Plan ahead.

---

*This document was written with the goal of building intuition, not just rules. Understanding why leads to better decisions than memorising what.*
