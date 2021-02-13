---
title: How to configure Proxmox
classes: wide
comments: true
categories:
  - proxmox
tags:
  - proxmox
  - hypervisor
  - ssh
---

## Overview

This guide will show you how to perform the initial configuration of a fresh install of Proxmox VE, version 6.3-3 at the time of writing.

This guide assumes knowledge of `vim` and basic Linux administration. It also assumes that you have installed the OS as a `ZFS` filesystem.

> It's absolutely critical that the PVE OS was installed as a `UEFI` OS. Usually in the boot menu in the `BIOS/UEFI` menu, it will show the installer USB disk as `UEFI: <Installer USB Drive>`

Sometimes you may boot into an `(initramfs)`, run these commands: `zpool import -R / rpool` then `exit`.

At this time, browse to: <https://<pve-ip>:8006>

## Verify UEFI Boot and OS Version

Check to see that `PVE` has in fact booted into `UEFI` mode:

``` bash
efibootmgr -v
```

Look for something along these lines:

``` bash
Boot0000* Linux Boot Manager	HD(2,GPT,<disk-uuid>)/File(\EFI\SYSTEMD\SYSTEMD-BOOTX64.EFI)
Boot0001* Linux Boot Manager	HD(2,GPT,<disk-uuid>)/File(\EFI\SYSTEMD\SYSTEMD-BOOTX64.EFI)
```

Check the installed version (Should read at least `6.3-X`):

``` bash
pveversion
```

## Enable No-Subscription Repository

[Proxmox VE No-Subscription Repository Documentation](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_package_repositories)

1. Login to the `pve` hypervisor:

    ``` bash
    ssh root@<pve-ip>
    ```

2. Enable the No-Subscription repositories

    ``` bash
    # vi /etc/apt/sources.list
    ```

    Before:

    ``` bash
    deb http://ftp.us.debian.org/debian buster main contrib
    deb http://ftp.us.debian.org/debian buster-updates main contrib

    # security updates
    deb http://security.debian.org buster/updates main contrib
    ```

    After:

    ``` bash
    deb http://ftp.us.debian.org/debian buster main contrib
    deb http://ftp.us.debian.org/debian buster-updates main contrib

    # PVE pve-no-subscription repository provided by proxmox.com,
    # NOT recommended for production use
    deb http://download.proxmox.com/debian/pve buster pve-no-subscription

    # security updates
    deb http://security.debian.org buster/updates main contrib
    ```

    Save and exit.

3. Disable Proxmox VE Enterprise Repository

    ``` bash
    vi /etc/apt/sources.list.d/pve-enterprise.list
    ```

    Before:

    ``` bash
    deb https://enterprise.proxmox.com/debian/pve buster pve-enterprise
    ```

    After:

    ``` bash
    # deb https://enterprise.proxmox.com/debian/pve buster pve-enterprise
    ```

    Save and exit.

4. Run update and install basic tools

    ``` bash
    apt update

    apt update && apt install vim tmux parted
    ```

5. Upgrade PVE OS Packages

    ``` bash
    apt upgrade

    apt dist-upgrade
    ```

## Nested Virtualization

[Enable Nested Hardware-assisted Virtualization](https://pve.proxmox.com/wiki/Nested_Virtualization#Enable_Nested_Hardware-assisted_Virtualization)

> Make sure that there are no running VMs at this time.

1. Check to see if it has been enabled. At this point it should show `N`

    ``` bash
    cat /sys/module/kvm_intel/parameters/nested
    ```

2. Add the Nested Virtualization Kernel Module

    > Only run one of these commands below

    If you have an Intel processor:

    ``` bash
    echo "options kvm-intel nested=Y" > /etc/modprobe.d/kvm-intel.conf
    ```

    If you have an AMD processor:

    ``` bash
    echo "options kvm-amd nested=1" > /etc/modprobe.d/kvm-amd.conf
    ```

3. Load the Kernel Module

    ``` bash
    modprobe -r kvm_intel
    modprobe kvm_intel
    ```

4. Verify that it has been enabled. Should show `Y`

    ``` bash
    cat /sys/module/kvm_intel/parameters/nested
    ```

## Install Cloud-Init

[Cloud-Init](https://pve.proxmox.com/wiki/Cloud-Init_Support)

At this point, let's just install `cloud-init`, more information can be found in another post.

1. Install package

    ``` bash
    apt install cloud-init
    ```

## (Optional) ZFS Imports

Most `PVE` hypervisors are configured to use disks as VM/Container storage, but in this case we want to use additional `ZFS Pools` for data storage that can be mounted/shared with VMs and Containers. 

1. If no pools have been created, follow the instructions [here.](https://pve.proxmox.com/wiki/ZFS_on_Linux#_zfs_administration)
2. Import pools

    ``` bash
    zpool import -f <pool-name>
    ```

3. Enable `ZFS` Scan and import services on boot 

    Enable `zfs` import services:

    ``` bash
    systemctl enable zfs-import-scan.service
    systemctl enable zfs-import.target
    ```

    Update `initramfs`:

    ``` bash
    update-initramfs -u -k all
    ```

## (Optional) Install NFS Server

Most guides recommend using an external NFS server or appliance OS, which is preferred if running as a `PVE Cluster` as it allows for migration across hypervisors.

This guide configures the hypervisor as an all-in-one server.

Here's how to install and configure NFS Server:

1. Install NFS Server

    ``` bash
    apt update

    apt install nfs-common nfs-kernel-server
    ```

2. Configure Server

    ``` bash
    vim /etc/exports
    ```

    Add NFS exports (shares) to end of the file:

    ``` bash
    <share-path> <network, e.g. 192.168.1.0>/255.255.255.0(rw,no_root_squash,subtree_check)
    ```

3. Export and check setup:

    Export all shares:

    ``` bash
    exportfs -a
    ```

    List all shared mounts:

    ``` bash
    showmount -e <ip-of-pve-host>
    ```

## (Optional) Add Networking Tools

If there is a need to add additional networks to the `PVE` hypervisor, install these packages

1. Install these packages

    ``` bash
    apt install ifupdown2 tcpdump
    ```

2. Configure secondary network bridge

    Navigate to `<pve-hostname>, System, Network, Create, Linux Bridge`

    Add the following:

    ```
    IPv4/CIDR: <subnet-to-add>/24
    Bridge ports: <bridge-interface>
    ```

    Be sure to **NOT** add the gateway

3. Click `Apply Configuration`

## (Optional) Install Dark Theme

[Dark Theme](https://github.com/Weilbyte/PVEDiscordDark)

Install the Dark Theme like this:

``` bash
wget https://raw.githubusercontent.com/Weilbyte/PVEDiscordDark/master/PVEDiscordDark.py

python3 PVEDiscordDark.py

[~] PVEDiscordDark Utility

[I] Install theme
[Q] Exit

>? I

[~] PVEDiscordDark Utility

Backing up index template file..
Downloading stylesheet..
Downloading patcher..
Applying stylesheet and patcher..
Downloading images [22/22]..

Theme installed successfully!
Press [ENTER] to go back. <ENTER>

[~] PVEDiscordDark Utility

[I] Install theme
[U] Uninstall theme
[Q] Exit

>? Q
```

## Final Steps

After performing all the selected steps above, reboot the machine and verify that everything is set up correctly.

``` bash
reboot
```