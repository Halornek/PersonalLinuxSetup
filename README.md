# PersonalLinuxSetup

This document is primarily designed as a failsafe to restore my Linux Desktop OS setup in case I ever reach an unrecovable state.

## Disclaimer

This guide is presented as is. Some of the instructions in this guide cover things such as resetting a drive's partition table which could lead to a loss of data. Anytime these are discussed please be very careful with how you proceed. I take no responsibilities for loss of data.

## System Specifications
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

### Enable CPU Virtualization

Boot into your systems UEFI and search for the CPU virtualization flag. This varies depending on vendor, but can typically be found in some kind of advanced settings menu.

### Enable IOMMU

IOMMU (Input-Output Memory Management Unit) will be required during later steps to properly isolate and passthrough system components.

This feature is also typically found under some kind of advanced settings menu.

### Disable Secure Boot

Secure boot can cause some issues with booting the guest Operating system and properly passing through physical components via the hypervisor.

This feature is typically found under the boot menu.

### Disable Legacy Boot

Legacy (BIOS) boot can also cause a large deal of issues with passing through graphics cards in particular.

## Configure Winders

This section is only necessary if Windows is already installed and the goal is to convert it to a guest OS. Please note that this method can cause issues with licensing as Windows will not see the guest OS as running on the same hardware.

### Ensure UEFI Boot

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

### Force UTC

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

### Disable Fast Startup

*Can be ignored if using VM*

Windows default behavior when instructed to "Shut Down" is to instead put the drive into a deep sleep state. This makes the drive write protected and would prevent your Linux OS from writing anything to the drive if you used "Shut Down" in Windows instead of "Restart".

This can be avoided by opening Command Prompt/Powershell as admin (Shift right click on the start menu) and entering:
```
powercfg -h off
```

## Prepare the Linux Drive

Ideally, the specified drive should be in an un-initialized state. This can be done using either Windows or Linux.

### Important Note

Be very careful selecting which drive to wipe. These commands are destructive and if you select the wrong drive you could lose some or all of your data.

### Windows (Diskpark)

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

### Linux (wipefs)

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

#### Important Note

If you run `wipefs` from the live installation environment it is advised to restart before continuing to the installion.

## Install Arch

### Create the Installation Media

You can find a .iso file for Arch Linx on their [downloads](https://archlinux.org/download/) page.

I normally just grab the file from their `pkgbuild.com` mirror. It should be just after `Checksums` listed under `Worldwide`. Once on the Index page you will want to grab the first file ending in `-x86_64.iso`.

Use an application such as [Rufus](https://rufus.ie/en/) (Windows), [Etcher](https://etcher.balena.io/) (Windows & Linux), or [Ventoy](https://www.ventoy.net/en/index.html) to write the .iso to a USB drive and use your boot menu (Check your motherboard documentation) to boot into the live installation environement.

### Setup the Drive

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

#### Configure EFI Partition

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

Now we enter the EFI System partition type (If you need a list, press `L` to list the various types. You can press `Q` once you are done reviewing to make your selection):
```
1
```

#### Configure LVM Partition

For the remainder of the drive we are going to use what is called `lvm`. LVM is a technology that allows you to create virtual partitions under a physical partition. There are a number of benefits of this, namely the ability to resize virtual partitions more easily than physical partitions.

To start, we are going to create a new partition:
```
n
```
We are going to press `enter` 3 times to default to partition number `2`, the newest starting sector, and the farthest ending sector.

Once the partition is created, we're going to assign a partition type to it like we did with the EFI partition.
```
t
```
This should default to the newest created partition number, `2`. Simply hit `enter`. Now we need to specify the type.
```
44
```
Do note that this value may change. It is best to press `L` to list the various types and locate `Linux LVM` to ensure it is correct.

#### Write to the Drive

Once you are done creating partitions we should review them:
```
p
```
This will list your partitions for the current drive. It should look something like:
```
Device           Start        End    Sectors  Size Type
/dev/sdb1         2048    1026047    1024000  500M EFI System
/dev/sdb2      1026048 1953523711 1952497664  931G Linux LVM
```
Do note worry if the start, end, sectors, and size match to the above table. The import thing is to have two partitions, `1` for the EFI partition and `2` for the LVM partition.

If this is correct, we want to issue a write command.
```
w
```
This should exit `fdisk` with the changes written to the disk.

#### Format Partitions

Now that the partitions are created we want to format them with a file system.

##### EFI Partition

To format a partition we are going to use `mkfs` (Make FileSystem) with the format FAT32:
```
mkfs.fat -F32 /dev/sdb1
```
As a reminder, `/dev/sdb` is our physical drive, while the `1` stands for the partition number.

##### LVM Partitions

For the remaining partitions, we're going to use `LVM`, a technology that lets us create and manage virtual partitions within a physical partition.

###### Physical Volume

To start with, we want to create the physical volume group:
```
pvcreate --dataalignment 1m /dev/sdb2
```
We are targeting partition `2` since that is what we partitioned under LVM earlier.

###### Volume Group

Now that we have a physical volume ready, we need to create a volume group on that physical volume. A volume group is a container for the logical volumes (virtual partitions) we will be using later. To do so:
```
vgcreate vga1 /dev/sdb2
```
We are now create the volumegroup `vga1`. This can be named whatever you want, I just use `vga` to keep things similar to a device such as `/dev/sdb`, with `sd` being "Sata Disk" and `b` being the disk identifier. In this case, `vg` stands for "Volume Group" and the `a` is the identifier I use for the first volume group. This is how I personally decide to set things up, but can use just about any name, so long as it's not part of a naming format such as `sda` or `nvme0n1`.

###### Logical Volumes

Now that our volume group is ready we can start splitting it up in the final partitions (Logical Volumes) our operating system will use, namely `root` and `home`, starting with `root`:
```
lvcreate -L 120GB vga -n lv_root
```
To break this down, we are using `lvcreate` with the `-L` flag to define a specific size of `120GB` under the volume group `vga` and using the `-n` flag to name the logical volume `lv_root`. This creates the final partition `/dev/vga/lv_root`.

Next we create the `home` partition.
```
lvcreate -l 100%FREE vga -n lv_home
```
Here we are once again using `lvcreate` with the `-l` flag to define a relative size of `100%FREE` under the volume group `vga` and using the `-n` flag to name the logical volume `lv_home`. This creates the final partition `/dev/vga/lv_home`.

###### Active the Volume Group

Now before we can format and use the logical volumes like a normal partition. To do so, we first need to enable the `dm_mod` (Device Mapper) module. The easiest way to do so is with `modprobe`:
```
modprobe dm_mod
```
This will allow us to run `vgscan` to scan all available disks for any `LVM` physical modules and volume groups.

Once this locates the volume group, we can activate it with:
```
vgchange -ay
```
The `-a` parameter tellss the system to set the active status of the specified (Or in all case, all) volume group(s) to the specified value (In this case, `y` for active).

###### Format the Logical Volumes

Now that the logical volumes are active and mapped, we can start formatting them for use by Linux and mounting them. To start with the `root` partition:
```
mkfs.ext4 /dev/vga/lv_root
```
Here we are using `mkfs` for format the device `/dev/vga/lv_root` (Our logical root partition) with the `ext4` file system. There are other file systems such as `btrfs` that have some benefits, but `ext4` is the general standard so I choose to stick wtih that.

As for the home partition it's fairly straightforward:
```
mkfs.ext4 /dev/vga/lv_home
```

#### Mount the Logical Volumes

To continue, we now need to understand the basics of "mounting" a storage device. In Windows, this can be seen as designating a drive letter to access the device. Linux does not use drive letters and instead allows the user to define a "mount point" where the device and all of it's files are attached on the base filesystem.

To mount the root partition:
```
mount /dev/vga/lv_root /mnt
```
What this does is mount the logical `root` volume to the mount point of `/mnt` in the live environment. The thing to remember here is that the live USB we are running off of is it's own Linux OS with a filesystem in place. To access and manipulate the drive we plan to install Arch onto we are mounting the logical partition underneath the `/mnt` directory.

Next we need to understand what a `home` partition/directory is. On most Linux systems, `/` is the root of the file system where various other folders will be mounted under. One of these is the `/home` folder. For security and convenience purposes, it's fairly standard for the `/home` folder to be it's own partition. Since we mounted the `root` partition at `/mnt`, we need to mount our home partition at `/mnt/home`.

However, that folder does not exist yet, so we need to create it:
```
mkdir /mnt/home
```
The `mkdir` (Make Directory) command is pretty straightforward. This just creates a directory at the target location. Now we can mount our home partition:
```
mount /dev/vga/lv_home /mnt/home
```
Don't worry about the fact that everything is under `/mnt`. This will all be cleared up in an upcoming section.

#### Generate fstab File

Now that we have the drives mounted we want to create the `fstab` file (This file is normally located at `/etc/fstab`). This file is what Linux uses to know what partitions to mount and where to place them in the filesystem.

This can be done using the `genfstab` command:
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```
What we are doing here is telling `genfstab` to grab device information for any partition listed underneath of `/mnt` including their UUIDs (Universal Unique Identifier) with the `-U` option. `-p` specified to ingore the `pseudofs` mounts, which is a default behavior. We aren't going to discuss `pseudofs` here.

However, on it's own `genfstab` only prints information to the console and does not actually modify any files. To fix this, we are "piping" the output of our `genfstab` command with `>>` to the file `/mnt/etc/fstab`.

To verify that everything looks correct, we are going to use `cat` to read the file we created:
```
cat /mnt/etc/fstab
```
This should print out a result similar to below:
```
# /dev/mapper/vga-lv_root
UUID=be8a58df-1ea1-47a9-8443-e0c2e62b4fdf       /               ext4            rw,relatime     0 1

# /dev/mapper/vga-lv_home
UUID=c11e490e-5f75-4fa1-9883-2b1a8947ee01       /home           ext4            rw,relatime     0 2
```
The actual UUIDs do not matter. These should be unique to your system. What should matter is that the `lv_root` partition is mounted under `/` with the permissions of `0` `1`, while the `lv_home` partition is mounted under `/home` with the permissions of `0` `2`. It isn't important to understand everything else in this file at this time.

### Starting the Install
