# Windows11 - ZFS - Setup and Benchmark

I've run benchmarks on Windows VMs on different block size configurations and with primarycache=all vs primarycache=metadata.

**Blocksize**
 - 16k
 - 64k
 - 128k

---

ZFS offers very good performance, but this is extremely workload based. So you can't run a banchmark and think that you will have the same performance in addition to your entire system. this is just a demonstration so that you can see the difference between the different blocksize and the difference between primarycache=all vs primarycache=metadata

---

## **Setup**
The windows machines are set up as follows:
 - Hide KVM and Hypervisor Flags for Windows
 - Installed offline, all VM is created with an offline account.
 - ReviOS is installed before Windows itself has time to receive its first update.
 - Windows terlementry - OFF
 - Windows Defendrer - OFF
 - ReviOS "Hacks" ON

This will make the optimal VM setup according to my tests.

---

I myself believe that ZFS snapshot is better to use than Windiws Defender as a precaution against possible viruses. this is entirely dependent on what you use your VM for, and I personally believe that Defender is just one of the major telementry systems Microsoft uses. decide for yourself what you think is right.

---

## ZFS Setup

For these tests I have used two of Western Digital 500GB nvme gen3 in ZFS stripe. When running a vm it is always good practice not to use the same drive that your host is running on.

Zpool configation:
```bash
sudo zpool create -f \
        -o ashift=12 \
        -o autotrim=on \
        -o feature@large_dnode=enabled \
        -O redundant_metadata=most \
        -O dnodesize=auto \
      nordix-vm /dev/disk/by-id/nvme-WDC_PC_SN720_SDAPNTW-512G-1006_1947BE451213_1 /dev/disk/by-id/nvme-WDC_PC_SN720_SDAPNTW-512G-1006_1947BE451213
```

Zvol configuation:

1. blocksize=16k
```bash
sudo zfs create -V 200G \
      -o volblocksize=16K \
      -o compression=lz4 \
      -o sync=disabled \
      -o redundant_metadata=most \
      -o logbias=throughput \
      -o primarycache=all \
      -o secondarycache=none \
      -o checksum=fletcher4 nordix-vm/zvol-win11-16k
```

2. blocksize=64k
```bash
sudo zfs create -V 200G \
      -o volblocksize=64K \
      -o compression=lz4 \
      -o sync=disabled \
      -o redundant_metadata=most \
      -o logbias=throughput \
      -o primarycache=all \
      -o secondarycache=none \
      -o checksum=fletcher4 nordix-vm/zvol-win11-64k
```

3. blocksize=128k
```bash
sudo zfs create -V 200G \
      -o volblocksize=32K \
      -o compression=lz4 \
      -o sync=disabled \
      -o redundant_metadata=most \
      -o logbias=throughput \
      -o primarycache=all \
      -o secondarycache=none \
      -o checksum=fletcher4 nordix-vm/zvol-win11-32k
```



