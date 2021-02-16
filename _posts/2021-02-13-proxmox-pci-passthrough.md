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
    1. Select `VirtIO-GPU` for `Graphic card:`
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

1. Select `Console` on the left, then click `Start` in the upper right.
2. As soon as you see the `Press any key to boot from CD or DVD..`, Hit `ENTER` on your keyboard. If you miss it, reboot and try again.
3. Once the Windows installer loads, fill out all the details until you hit `Where do you want to install Windows?`, then hit `Load driver`.
        ![Load driver](https://drive.google.com/uc?id=1h_4gXrxo9m1p3Cb3-cQGAeK7Xnbt3_2s)
4. Click `Browse`, then `Browse` again, double click on the `virtio-win-0.1.190` disk under `This PC`, then scroll down and click on `vioscsi`, then `w10`, then `amd64`, then hit `OK`. Hit `Next` until you get back to the install menu. **NOTE:** If you used the non-Intel network driver, the driver can be found here as well.
        ![Select driver](https://drive.google.com/uc?id=1poQubdeEvg5oUmsZBTkC6NzwYGCO5i-S)
        ![RedHat driver](https://drive.google.com/uc?id=1wsfbcjUcM2bdaNmHVPKExiM0YvOH77w-)
5. Now that the disk is found, select `Drive 0 Unallocated Space` and hit `Next`.
6. Finish the Windows install.
7. On first Windows boot while going through personalization settings, be sure to DISABLE all the settings on `Choose privacy settings for your device`, they love to track you...

### Configure Windows and install GPU drivers

Perform the following:

1. Make sure that a password is set for this user. Select Start, Settings, Accounts, Sign-in options. Under Password, select the Change button and follow the steps.
2. Enable `Remote Desktop`. Select Start, Settings, System, Remote Desktop. Use the slider to enable Remote Desktop.
3. Connect to host using `RDP`.
4. Install the `Qemu Agent`. Click on `Windows Explorer`, `This PC`, `virtio-win-0.1.190`, `guest-agent`, then install `qemu-ga-x86_64`.
5. Open `Device Manager` by right-clicking the Windows logo (start button) and selecting `Device Manager`. Once open, install all the missing drivers for the `Other devices/PCI *`. Right-click each one and select `Update driver`, then `Browse my computer for drivers`, Click `Browse` then select the `virtio-win-0.1.190` disk under `This PC` and hit `Next`. Install the drivers until there are no more exclamation drivers under `Other devices`. It should look like this when complete:
        Before:
        ![Install other driver](https://drive.google.com/uc?id=1bq9PW78fBlca6gIpP6Xyvb3fXiM8jCX4)
        After:
        ![After other driver install](https://drive.google.com/uc?id=1-MmnlalRWa8oH4ZiuY54SfPvjxQqouMh)
6. Download and install the Nvidia drivers. Download the drivers from this [link.](https://www.nvidia.com/Download/index.aspx). Install the drivers, but don't install the `GeForce Experience`.
7. Shut down the VM through Windows.
8. Select `Hardware`, click `PCI Device`, then enable all check boxes, i.e. `Primary GPU`.
9. Start up the VM.
10. Verify that the GPU is working properly. Right-click start menu, and hit `Device Manager`, if it's working you won't see an exclamation point on the GPU. Open up the `Display adapters` and right-click the GPU and 
        
    Properly working GPU (No `Code 43`):
    ![Device-manager-GPU](https://drive.google.com/uc?id=1H5IA4Bdxb6DPQI1dbAnTWe67V9OPfZ2S)
    
    Check `Task Manager` too:
    ![Task-Manager](https://drive.google.com/uc?id=1vZ0-kk_tMrigCx6z4mbQ0Q2L5wLug199)

11. If everything is working properly, you can connect the GPU to an external display. You may have to reboot the VM, but once it comes up, you should see the external display show video. If you don't, or if you only see the background, you may have to set the display to: `Duplicate these displays`. I'd also recommend setting the resolution to `1920 x 1080`. You'll get better gaming performance and will be able to have a better streaming experience.
12. Last, but not least, set a static IP address so that it's easier to connect to this host. If you're ever in a situation where you don't know the IP, you can click on the VM on the left side, hit `Summary` and you should see the VM's IP under the `IPs` section.

If you made it this far without issues, congratulations!

## Configure remote gaming

This section will likely vary depending on how you want to use this VM. Instead of repeat steps from other guides, I'll just link them here.

### Configure streaming

I'd strongly suggest you set up streaming so that this host can be used remotely. This is a great guide put together by the extremely helpful folks at `serverbuilds.net`.

This guide was built for `UnRAID`, which we aren't using here, but the streaming part of this guide directly applies here. Go ahead and follow the instructions outlined [here.](https://forums.serverbuilds.net/t/guide-remote-gaming-on-unraid/4248/11)

I'd recommend installing the following:

* [Parsec](https://parsec.app/)
* [Rainway](https://rainway.com/)
* [Other packages from Ninite](https://ninite.com/)

### Configure VM as a physical host

If you plan to use this VM as a physical machine, there are a few things to consider. It might be helpful to set this VM to start on `Proxmox` boot, click on the VM on the left, hit `Options`, double-click `Start at boot` and check the box.

In order to use this VM, you'll need to have a way to hook up a keyboard and mouse. I'd suggest passing through an entire USB port so that you can attach a USB hub to connect a series of devices. To do this, be sure the VM is shutdown before you do anything, then click the VM on the left-hand side, click `Hardware`, click `Add`, then select `USB Device`, click the option `Use USB Port` and select a port. Once it has been attached, boot the VM. You should be able to use those newly attached devices.

If you use an external monitor, or TV, I'd recommend using an HDMI port because you'll be able to get audio from the video card so that you don't have to install an external audio card. You could also use a USB audio card, but it's up to you.

More documentation for USB passthrough can be found [here.](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_usb_passthrough)

## Troubleshooting

* PCI Passthrough errors/issues
  * Re-check all the setup and verification steps above.
  * Be absolutely sure that the system booted via `UEFI`.
  * Make sure that `Display` is set to `VirtIO-GPU`.
  * Also verify that the resolution is set properly for you displays.
* GPU `(Code 43)` Error
  * This can be a real tough one, with an absolute TON of potential issues that would cause this, so be patient and look over your settings.
  * First try all the steps in the bullet-points above.
  * Try to uninstall and re-install the drivers for the GPU. Here's some helpful resources for this option [here.](https://www.crazywebstudio.co.th/2019/how-to-solve-code-43-nvidia-gtx10xx-or-rtx20xx-gpu-problem/)
  * Make sure that your video is new enough to support PCI passthrough.
  * Double-check your hardware for compatability.
  * Re-verify that `VT-d` has been enabled in the `UEFI/BIOS`.
  * Be absolutely sure that your kernel has loaded the `iommu` settings.

If all else fails, check out the resources below. Take your time and check off each item as you verify it. It's truly shocking how one tiny setting being off can cause it all to fail.

## References

* [Proxmox VE 6.3-3 PCI(e) Passthrough Guide](https://elijahliedtke.medium.com/home-lab-guides-proxmox-6-pci-e-passthrough-with-nvidia-43ccfb9424de)
* [Proxmox VE Alternative install with Systemd-boot](https://hackmd.io/@edingroot/SkGD3Q7Wv#Guest-OS-Config)
* [Proxmox VE Official PCIe Passthrough guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)
* [Proxmox VE 6.1 Ultimate Beginners Guide to GPU Passthrough](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/)
* [Remote Gaming Guide](https://forums.serverbuilds.net/t/guide-remote-gaming-on-unraid/4248/11)