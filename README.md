# Nordix VM

- [**ZFS, ARC, zvol and Virtual Machines**](zvol-and-Virtua-Machines.md) - A conceptual ZGuide to understanding how they work together
- [**Windows11 VM on ZFS, Benchmarks**](ZFS-Windows-Benchmarks.md) - Windows on ZFS: Zvol Test results for different blocksize, and the difference between primarycache=all VS primarycache=metadata
- [**ZGuide: Window11 VM on ZFS Zvol**](win11-zfs-setup.md) - Nordix Zguide: Setup windows vm on zfs zvol
----
## To do project - VM so simple it's impossible to fail 

**I have an idea to create a system for virtual machines, an installation system, this system is like a complement to virtmanager.**

A guided GUI system, where you only need to enter a few choices
> - OS: Which operating system you gonna install
> - ISO: Path to your iso file
> - Zvol - size and where to install the zvol, The standard suggestion will be "nordix
> - CPU: how many you want to give to the VM
> - RAM: how much you want to give the VM

 - What happens behind the scenes is that you apply ready-made optimized virsh templates, that you run nordix tools for bridged network and that you activate ssh
 - checking that the correct services are enabled for libvirt, that this user is in the groups `wheel audio disk input kvm storage video qemu libvirt`
 - A system service that creates a desktop entry for this VM, so that it is added to your display manager, this allows you to start your VM directly at login and then with minimal overhead from the host
   
You can also use zfs clones here if you want to create a dedicated dataset to boot for just VM execution, for those who want to run their VMs with gpu passthrough so as not to have to change the standard system's root dataset with virtio driver and lock gpu to vifo, being able to make it a dedicated dataset is then very convenient.

> _Later I also figured out how one could create a very efficient share system between host and VM with zvol and some simple scripts to synchronize data between zvol and share directory, to only sync data if this zvol is not mounted on the guest._
---
My goal is to create as simple a VM system as possible so that you will never feel like you need anything other than Nordix, that VM should be so simple and integrated into Nordix that the display manager feels like a regular dualboot menu.

