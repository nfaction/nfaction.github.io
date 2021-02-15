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

Verify loaded kernel `cmdline`:

``` bash
$ cat /proc/cmdline
initrd=\EFI\proxmox\5.4.73-1-pve\initrd.img-5.4.73-1-pve root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
```

Verify that `IOMMU` is enabled and `Virtualization Technology for Directed I/O`:

``` bash
$ dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
[    0.013784] ACPI: DMAR 0x000000007E2CAB18 000130 (v01 A M I  OEMDMAR  00000001 INTL 00000001)
[    0.155343] DMAR: IOMMU enabled

...

[    1.474961] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

Verify `vfio`:

``` bash
$ dmesg | grep -i vfio
[   16.913623] VFIO - User Level meta-driver version: 0.3
```

Look for GPU:

``` bash
$ lspci -nn | grep NVID
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206 [GeForce GTX 960] [10de:1401] (rev a1)
03:00.1 Audio device [0403]: NVIDIA Corporation GM206 High Definition Audio Controller [10de:0fba] (rev a1)
```

Look for `IOMMU_groups` and `grep` using hardware address above:

``` bash
find /sys/kernel/iommu_groups/ -type l | grep 03:00
/sys/kernel/iommu_groups/25/devices/0000:03:00.0
/sys/kernel/iommu_groups/25/devices/0000:03:00.1
```

If you have similar output, congratulations! You have sucessfully passed through your PCI(e) devices. If for some reason you don't show this output, go back and re-verify all the above steps.

## Install Windows

### Get Installer ISOs

Download the following:

* [Win10_20H2_v2_English_x64.iso](https://www.microsoft.com/en-us/software-download/windows10ISO)
* [Download VirtIO drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso)

Upload them to the `Proxmox VE` host:

1. Login to the web UI: `https://<pve-ip>:8006`
2. Click on hypervisor hostname on the left, then click `local (<pve-hostname>)`, then `ISO Images`, then click `Upload`, browse and upload the two files above.

You should see something like this after you are done:

![ISO Images](https://drive.google.com/uc?id=1XLAQ-zcDcYFLW68BMGLrRYmtWKTgvONl)

### Create Windows VM

1. Click `Create VM` in the upper right.
2. `General`
    1. Set `VM ID:`. E.g. `111`, anything above 100 will do
    2. Set `Name:`. E.g. `windows-10`
    3. Click `Next`
3. `OS`
    1. Select `Win10_20H2_v2_English_x64.iso` as the `ISO Image:`, under `Use CD/DVD...`
    2. Select `Microsoft Windows` for `Type:`
    3. Select `10/2016/2019` for `Version:`
    4. Click `Next`
4. `System`
    1. Select `Default` for `Graphic card:`
    2. `VirtIO SCSI` for `SCSI Controller:`
    3. `Qemu Agent:`, Checked
    4. `OVMF (UEFI)` for `BIOS:`
    5. `q35` for `Machine:`
    6. `Add EFI Disk:`, Checked
    7. `local-zfs` for `Storage:`
    8. Click `Next`
5. `Hard Disk`
    1. `SCSI` for `Bus/Device:`
    2. `Write back` for `Cache`.
    3. `local-zfs` for `Storage:`
    4. `80` for `Disk size (GiB):`, or any size you prefer
    5. Check the following: `Discard`, `SSD emulation:`, `IO thread:`, `Skip replication:`.
    6. Click `Next`
6. `CPU`
    1. `4` for `Cores`, `1` for `Sockets:`
    2. `Type:`,  this one is the most important setting. Set this to `IvyBridge` if you have this family of processors. If you don't know, see the mast list here: [Intel CPU Architechture List](https://en.wikipedia.org/wiki/List_of_Intel_processors#64-bit_processors:_Intel_64_%E2%80%93_Sandy_Bridge_/_Ivy_Bridge_microarchitecture)
    3. `Enable NUMA`, Checked.
    4. `pcid`, `+` in the bottom under `Extra CPU Flags:`
    5. Click `Next`
7. `Memory`
    1. `8192` for `Memory (MiB)`, or any size you prefer. Just multiply the number of GB by 1024. For example: `8 * 1024 = 8192`.
    2. Click `Next`
8. `Network`
    1. `Intel E1000` for `Model:`. This one could vary for you. However, I use this one because Microsoft has drivers for this. If you use anything else you may have to install drivers when installing Windows from the ISO.
    2. Click `Next`

You should see something like this on the `Confirm` page.

![Windows 10 VM Confirm](https://drive.google.com/uc?id=1T34f_WizI40a4fr72-TgHZnSy5X06nJz)

Click `Finish`.

### Add hardware before first boot

Perform the following:

1. Click on the VM ID you created, in this example `111 (windows-10)` on the left side.
2. Click `Hardware`
3. Add a secondary DVD drive, Click `Add`, `CD/DVD Drive`, `Bus/Device: SATA`, `Storage: local`, `ISO Image: virtio-win-0.1.190.iso`, `Create`.
4. Add the GPU, Click `Add`, `PCI Device`, Select the GPU you wish to pass through, E.g. `0000:03:00.0`, Check all boxes but `Primary GPU`, Click `Add`

### Install Windows

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