---
title: How to configure PCI(e) passthrough on Proxmox VE
classes: wide
comments: true
author_profile: true
related: true
categories:
  - proxmox
tags:
  - proxmox
  - gpu
  - gaming
  - hypervisor
---

## Overview

This guide will show you how to configure PCI(e) Passthrough on Proxmox VE, version 6.3-3 at the time of writing. In this case, we'll be using a GPU as the passthrough device.

This guide assumes that the `Proxmox VE` OS was installed using `ZFS RAID1` or similar, knowledge of `vim` and basic Linux administration. The big difference between this type of install is that it uses `Systemd-boot` instead of the typical `GRUB` install. If you don't know `vim`, use `nano` instead.

If you are unsure of your settings, please refer to the setup guide post [here.](/proxmox/proxmox-install/)

Hardware used in this tutorial:

* Motherboard: Supermicro X9DRL-3F/iF
* CPU: 2 x Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz (2 Sockets, 32 Cores)
* RAM: 32GB (8 x 4GB) DIMM DDR3 1333 MHz ECC Memory
* OS Disks: 2 x 500GB SSDs
* Data disks: 9 x 3TB SATA Disks
* GPU: Nvidia GTX 960 SSC

**NOTE:** Although it's preferable to use a PCIe x16 slot, a PCIe x8 slot will actually work just as well. The only requirement for using a x8 is that it must be a "cut" slot, meaning that the slot is open in the rear where a larger slotted card can be used. You might notice that the motherboard I am using is this case exactly. Usually the only downside to running a GPU in this way will result in a few lost FPS, but that's about it.

## Pre-flight Checks

PCI(e) Passthrough depends on a number of requirements to work properly. Be absolutely sure that these items have been taken care of or you will waste countless hours.

Here are those strict requirements:

* A GPU that is `UEFI Boot` capable. Most cards equivalent to a Nvidia GTX 600 series or newer will work. However, I'd suggest using something newer like a GTX 900 series. AMD graphics cards will work just as well, but again, they must be a similar series.
* The `BIOS/UEFI` must be set to boot into the `Proxmox VE` OS in `UEFI` mode. PCI(e) passthrough requires this.
* Virtualization flags enabled in the `BIOS/UEFI`, specifically `VT-d` and `Virtualization`.

## Proxmox VE Host Configuration

[Official Proxmox VE PCIe Documentation](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)

### IOMMU Systemd-boot

[Systemd-boot Kernel Commandline Documentation](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysboot_edit_kernel_cmdline)

Add the `intel_iommu=on iommu=pt` settings to the `Systemd-boot` command line.

Edit the `/etc/kernel/cmdline`:

``` bash
vim /etc/kernel/cmdline
```

Before modification:

``` bash
root=ZFS=rpool/ROOT/pve-1 boot=zfs
```

After:

``` bash
root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
```

Update the `Systemd-boot` scripts:

```
pve-efiboot-tool refresh
```

### Kernel Modules

Add the required kernel modules:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Edit `/etc/modules`:

``` bash
vim /etc/modules
```

Before:

``` bash
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
```

After:

``` bash
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Refresh the `initramfs`:

``` bash
update-initramfs -u -k all
```

### Reboot PVE host

``` bash
reboot
```

### Verify IOMMU/PCI(e) Passthrough

``` bash

```

``` bash

```

``` bash

```

``` bash

```

``` bash

```



1. Enable EFI Boot
	1. Turn on Storage OPRom (Enables UEFI menus in boot)
2. Enable Virtualization and VT-d
3. Boot into OS
4. Follow steps here: https://elijahliedtke.medium.com/home-lab-guides-proxmox-6-pci-e-passthrough-with-nvidia-43ccfb9424de
5. Enable systemd boot settings from this example: https://hackmd.io/@edingroot/SkGD3Q7Wv#Guest-OS-Config
6. Works!
7. Setup parsec: https://forums.serverbuilds.net/t/guide-remote-gaming-on-unraid/4248/7

## Troubleshooting

* Code (43)
* Passthrough issues

## References

* [Proxmox VE 6.3-3 PCI(e) Passthrough Guide](https://elijahliedtke.medium.com/home-lab-guides-proxmox-6-pci-e-passthrough-with-nvidia-43ccfb9424de)
* [Proxmox VE Alternative install with Systemd-boot](https://hackmd.io/@edingroot/SkGD3Q7Wv#Guest-OS-Config)
* [Proxmox VE Official PCIe Passthrough guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)
* [Remote Gaming Guide](https://forums.serverbuilds.net/t/guide-remote-gaming-on-unraid/4248/11)