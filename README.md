# Disclaimer

This guide is presented as is. Some of the instructions in this guide cover things such as resetting a drive's partition table which could lead to a loss of data. Anytime these are discussed please be very careful with how you proceed. I take no responsibilities for loss of data.

# PersonalLinuxSetup

This document is primarily designed as a failsafe to restore my Linux Desktop OS setup in case I ever reach an unrecovable state.

## System Specifications:
```
CPU: Ryzen 9 5950X
Mobo: Gigabyte X570 Aorus Master
RAM: 2 x 16GB Trident Z Neo DDR4-3600 @CL16
GPU: Gigabyte Reference Radeon RX 6900 XT
SSD: (Windows): Sabrent Rocket NVMe4 1TB
SSD: (Linux): Crucial P5 NVMe3 1TB
SSD: (Shared): Crucial BX500 2.5" SATA III
PSU: Corsair RM1000x 80+ Gold
```

## Pre-Installation - UEFI

A couple settings need to be changed within the system UEFI firmware.

#### Enable CPU Virtualization

Boot into your systems UEFI and search for the CPU virtualization flag. This varies depending on vendor, but can typically be found in some kind of advanced settings menu.

#### Enable IOMMU

IOMMU (Input-Output Memory Management Unit) will be required during later steps to properly isolate and passthrough system components.

This feature is also typically found under some kind of advanced settings menu.

#### Disable Secure Boot

Secure boot can cause some issues with booting the guest Operating system and properly passing through physical components via the hypervisor.

This feature is typically found under the boot menu.

#### Disable Legacy Boot

Legacy (BIOS) boot can also cause a large deal of issues with passing through graphics cards in particular.

## Configure Winders

This section is only necessary if Windows is already installed and the goal is to convert it to a guest OS. Please note that this method can cause issues with licensing as Windows will not see the guest OS as running on the same hardware.

#### Ensure UEFI Boot

Use diskpart to check if Windows is installed under a GPT disk or an MBR disk. The easiest way to do this is to press Windows+R and run:
```
diskpart
```
Once `diskpart` has loaded, simply type:
```
list disk
```
This should spit out something similar to the below image.
```
Microsoft DiskPart version 10.0.19041.964

Copyright (C) Microsoft Corporation.
On computer: BD-VW-001

DISKPART> list disk

  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online          931 GB  1024 KB        *
```
If your primary boot disk has a `*` under `Gpt`, then Windows is already installed under UEFI boot.

If there is not a `*` under `Gpt` for your primary boot disk, then Windows is running under legacy (MBR) boot. You can convert it using the Windows tool [MBR2GPT](https://docs.microsoft.com/en-us/windows/deployment/mbr-to-gpt).

#### Force UTC

*Ignore if using VM*

Windows and Linux will often use two different methods to measure time. In a dual boot scenario this can cause some issues with clock syncronization when first booting Linux, then restarting into Windows.

Linux uses UTC, a universal time clock that adjusts for time zone in the OS.

Windows uses RTC, a local time clock that adjusts the UEFI clock to match local time.

If you are dual booting instead of setting up a Virtual Machine for Windows, you can force Windows to use UTC by making a registry edit. Run:
```
regedit
```
Then navigate to:
```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation
```
Right click the directory `TimeZoneInformation`, go to `New`, and select `DWORD (32-Bit) Value`.

Name the value `RealTimeIsUniversal`, set the value to `1`, and ensure the base is set to `Hexadecimal`.

#### Disable Fast Startup

*Can be ignored if using VM*

Windows default behavior when instructed to "Shut Down" is to instead put the drive into a deep sleep state. This makes the drive write protected and would prevent your Linux OS from writing anything to the drive if you used "Shut Down" in Windows instead of "Restart".

This can be avoided by opening Command Prompt/Powershell as admin (Shift right click on the start menu) and entering:
```
powercfg -h off
```

## Prepare the Linux Drive

Ideally, the specified drive should be in an un-initialized state. This can be done using either Windows or Linux.

#### NOTE:

Be very careful selecting which drive to wipe. These commands are destructive and if you select the wrong drive you could lose some or all of your data.

#### Windows (Diskpark)

First, open `Disk Management` (Either Shift+Right Click on the start menu or open the start menu and search for Disk Management) to locate the specific drive you want to install Linux on. Once you have the drive number, run (Win+R):
```
diskpark
```
This will open a command line utility where you can then list your disks:
```
list disk
```
You should receive a result that looks something like:
```
Microsoft DiskPart version 10.0.19041.964

Copyright (C) Microsoft Corporation.
On computer: BD-VW-001

DISKPART> list disk

  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online          931 GB  0 B            *
  Disk 1    Online          447 GB  0 B            *
  Disk 2    Online         1863 GB  0 B            *
  Disk 3    Online          117 GB  0 B            
```
In our case, we want `Disk 1', so we will select it:
```
select disk 1
```
Once the disk is selected, we can reset it:
```
clean
```
The drive is now reset and can be used to install Linux.

#### Linux (wipefs)

Ensure that you have the 'wipefs' package installed. This is included in most base distrobution packages (`core` for Arch Linux).

Identify the drive you wish to wipe.
```
sudo fdisk -l
```
In our case, the device is named `/dev/sdb`. Linux stores device data under the `/dev` directory where files and folders reference to the actual devices. `sd` stands for "Sata Device" while the last character `b` denotes drive b (This is not a drive letter). Partitions will appear like `/dev/sdb1`, where `sdb` idtenified the volume, and `1` identifies the partition.

Now that we know the device name we can reset it with wipefs:
```
sudo wipefs -a /dev/sdb
```
This will wipe all the device IDs used by an MBR or GPT volume.

##### Note:

If you run `wipefs` from the live installation environment it is advised to restart before continuing to the installion.

## Install Arch

#### Create the Installation Media

You can find a .iso file for Arch Linx on their [downloads](https://archlinux.org/download/) page.

I normally just grab the file from their `pkgbuild.com` mirror. It should be just after `Checksums` listed under `Worldwide`. Once on the Index page you will want to grab the first file ending in `-x86_64.iso`.

Use an application such as [Rufus](https://rufus.ie/en/) (Windows), [Etcher](https://etcher.balena.io/) (Windows & Linux), or [Ventoy](https://www.ventoy.net/en/index.html) to write the .iso to a USB drive and use your boot menu (Check your motherboard documentation) to boot into the live installation environement.

#### Setup the Drive

We are going to use `fdisk` for partition the drives and get them ready for formatting.

First, find the drive you are installing to:
```
fdisk -l
```
In our case, we will be installing to `/dev/sdb`. Please remember the identifier for the drive you will be using. NVMe drives may appear as `/dev/nvme0n1`.

Use `fdisk` to select the drive for partitioning.
```
fdisk /dev/
```
This will enter the `fdisk` utility on the targeted drive. We can now tell it to create a GPU (UEFI) partition table.
```
g
```
We now want to create a new partition.
```
n
```
It should now ask for the partition number. You can leave this blank and hit `enter` to default to `1`.

Next it will ask about the starting block. Once again, you can leave this empty and hit `enter` to use the first available block.

Now we want to specify the last block of the partition. This is supposed to be the EFI partition where your bootloader will sit, so it does not need to be very large. We're going to give it a size of 500 Megabytes.
```
+500M
```
Now that the partition is created and you are back in the `fdisk` utility we want to convert the file type to an EFI system partition. Type in:
```
t
```
Since there is only 1 partition currently on the drive, the `t` command used to change the partition type will select the first partition.
