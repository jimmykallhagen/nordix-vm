# Windows11 VM on ZFS, Benchmarks

## Benchmarks
- These VM runs on two WD nvme gen3 i ZFS stripe, with a dedicated zvol for a "real" nvme for the windows to run on
- These benchmarks is made with DDR5 6000mhz 64gb RAM

---

1. blocksize=16k and primarycache=metadata
![16k-metadata](screenshot/16k/Screenshot_2026-03-23_17-15-08.png)

2. blocksize=16k and primarycache=all
![16k-all](screenshot/16k/Screenshot_2026-03-23_17-05-05.png)

---

3. blocksize=64k and primarycache=metadata
![64k-metadata](screenshot/64k/Screenshot_2026-03-23_17-37-14.png)

4. blocksize=64k and primarycache=all
![64k-all](/screenshot/64k/Screenshot_2026-03-23_16-58-15.png)

---

5. blocksize=128k and primarycache=metadata
![128k-metadata](screenshot/128k/Screenshot_2026-03-23_19-16-39.png)

6. blocksize=128k and primarycache=all
![128k-all](screenshot/128k/Screenshot_2026-03-23_19-21-25.png)

---
