# Arch Install Instructions

## Target Audience

While this document is primarily designed as a failsafe to restore my Linux Desktop OS setup in case I ever reach an unrecovable state, the general target audience would be the entry/mid-tier Windows Power User. The "family tech support" that used to be able to do everything since they could easily navigate and configure Windows via GUI applications such as Control Panel and may be frustrated that Microsoft is slowly taking away these things and regulating them to more advanced utilities such as `gpedit` and `regedit`.

## Disclaimer

This guide is presented as is. Some of the instructions in this guide cover things such as resetting a drive's partition table which could lead to a loss of data. Anytime these are discussed please be very careful with how you proceed. I take no responsibilities for loss of data.

## System Specifications

Keep in mind that this guide was written with the following specifications as a base. You will at least want to know what type of drives you have in your system along with your memory capacity, CPU, and GPU.
```
CPU: Ryzen 9 5950X
Mobo: Gigabyte X570 Aorus Master
RAM: 2 x 16GB Trident Z Neo DDR4-3600 @CL16
GPU: Gigabyte Reference Radeon RX 6900 XT
SSD: (Windows): Sabrent Rocket NVMe4 1TB
SSD: (Linux): Crucial P5 NVMe3 1TB
SSD: (Shared): Crucial BX500 2.5" SATA III
SSD: (Install Drive): Crucial MX500 2.5" SATA III
PSU: Corsair RM1000x 80+ Gold
```

## Configure Winders

This section is only necessary if Windows is already installed and the goal is to convert it to a guest OS or dual boot. Please note that using Windows as a guest OS can cause issues with licensing as Windows will not see the guest OS as running on the same hardware.

### Ensure UEFI Boot

Modern computers have two different booth methods. The older method is often referred to as BIOS or "Legacy" boot. Legacy boot uses boot drives set up as MBR ("Master Boot Record"). MBR has a number of issues such as limiting the total usable size of a drive to 2.2TBs and not properly support encryption. Legacy boot can also cause some issues with passing through things like graphics cards to a VM.

The newer method is often referred to as UEFI (Unified Estensible Firmware Interface). Nearly all modern computers will use a UEFI boot with the operating system installed on a GPT (GUID (Globaly Unique Identifiers) Partition Table) drive. UEFI boot and GPT drives offer additional features and security. These should be used whenever possible.

If you are dual booting, it is very important that both Windows and Linux are using the same boot method. Since this guide is only written with UEFI in mind (It is possible to install Arch Linux under Legacy Boot, but I am not covering this as a majority of modern systems should be using UEFI) we will need to ensure that Windows is already running with UEFI boot before we disable the legacy boot method in the system firmware at a later step.

Most of the time newer Windows installs will use a UEFI boot method on a GPT (GUID (Globaly Unique Identifiers) Partition Table) drive. Since we are disabling legacy boot methods we need to ensure Windows is not installed on an MBR drive, which would be using a Legacy Boot method.

To start, use diskpart to check if Windows is installed under a GPT disk or an MBR disk. The easiest way to do this is to press Windows+R and run:
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

## Pre-Installation - UEFI

A couple settings need to be changed within the system UEFI firmware (Sometimes called the "BIOS").

### Disable Secure Boot

Secure boot can cause some issues with booting some Linux operating systems and allowing `grub` (The Linux Bootloader we will be using) to hand off the boot process to Windows in a dual boot environment. Secure boot offers very little actual security and is more of a tool for Microsoft to only allow "approved" operating systems to boot.

This feature is typically found under the boot menu.

### Disable Legacy Boot

Since we ensured/converted Windows to UEFI we can go ahead and disable legacy boot methods.

This feature is typically found under the boot menu.

## Prepare the Linux Drive

Ideally, the specified drive should be in an un-initialized state. This can be done using either Windows or Linux.

Only use one of the below methods to prepare the drive. If the drive is already in an uninitialized state (Recently purchased new) then you can skip this part. To clarify, this is not the same as formatting/partitioning the drive for use by Linux. This step is to remove the partition tables so that the setup process will be easier later.

### Important Note

Be very careful selecting which drive to wipe. These commands are destructive and if you select the wrong drive you could lose some or all of your data.

### Windows (Diskpart)

First, open `Disk Management` (Either Shift+Right Click on the start menu or open the start menu and search for Disk Management) to locate the specific drive you want to install Linux on. Once you have the drive number, run (Win+R):
```
diskpart
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
In our case, the device is named `/dev/sdb`. Linux stores device data under the `/dev` directory where files and folders reference to the actual devices. `sd` stands for "Sata Device" while the last character `b` denotes drive b (This is not a drive letter). Partitions will appear like `/dev/sdb1`, where `sdb` idtenified the drive, and `1` identifies the partition.

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

Now that our drives are in place we can start installing everything necessary to boot our Arch Linux system.

To start with we are going to explain Arch's primary package manager, `pacman` (Litterally "Package Manager"). `pacman` is a simple package manager that accepts basic options and a list of package names to perform actions. We will go further in depth later, because first we need to install `pacman` to our in-progress installation. This is done by installing the `base` package using `pacstrap`:
```
pacstrap /mnt base
```
This will install the `base` package group treating `/mnt` as if it was `/`. This is what is called a `chroot` (Change Root).

Speaking of `chroot`, we are not going to change our shell's root into the in-progress installation with:
```
arch-chroot /mnt
```
This should change how your command line appears because it is now running from within your in-progress installation. Remember, all of the files for your in progress installation are underneath of `/mnt` on the live environment.

#### Installing the Kernel

From here we can start installing packages. The first of which we will install is the kernel and it's headers.
```
pacman -S linux linux-headers
```
This will install the mainline Arch Linux kernel and it's relative headers, those being an application that acts as a middleman between internal kernel components and userspace applications.

##### Alternative Kernels

It is important to note that you can install different versions of the kernel. A quick caveat to this however. If you are using an Nvidia GPU stick to either the the mainline `linux` kernel (And it's relative `linux-headers`) or the long term support `linux-lts` kernel (and it's relative `linux-lts-headers`). These are the only kernels that offer proper update support for Nvidia's proprietary drivers. It is possible to create what is called a `hook` within `pacman` to reinstall the drivers every kernel update, but this is not recommended or being covered.

The main options are:
`linux` and `linux-headers`: The mainline kernel that is maintained by the Arch Linux developers.
`linux-lts` and `linux-lts-headers`: A much slower mainline kernel that stays on an `LTS` (Long Term Support) branch of the Linux Kernel. This option is designed for stability and may miss out on new features or hardware support. It still does receive frequent security patches, however.
`linux-hardened` and `linux-hardended-headers`: A very strict and locked down kernel designed for "mission critical" systems. This kernel comes with additional security patches and features that would not normally be utilized by the standard desktop user.
`linux-zen` and `linux-zen-headers`: A more community maintained kernel. This is built off the mainline kernel with additional patches that might enable better hardware and feature support (Such as the `OpenRGB` kernel patch for i2c controllers in order to access DRAM and motherboard RGB features).

#### Install Packages

Next we want to install some additional packages that should be pretty standard on most Linux systems.
```
pacman -S linux-firmware nano base-devel openssh
```
`linux-firmware` adds support for most general hardware. This is required for a graphical environment.
`nano` is a simple text editor we will be using to modify some configuration files later.
`base-devel` includes extra packages from `base` that offer some more tinker friendly features.
`openssh` allows you to enable the SSH daemon, letting you remote into a shell environment from another device. This is very useful when troubleshooting something like GPU drivers or display environments. You will want to enable the SSH daemon with:
```
systemctl enable sshd
```
`systemctl` is used to control `systemd` (Our init system, will be covered later) services. This can `start`, `stop`, `enable`, `disable`, or `restart` services. In this case, we are flagging the `sshd` (SSH daemon, or server) to run at boot as a background service.

Additionally, we want to install some networking tools.
```
pacman -S networkmanager wpa_supplicant wireless_tools netctl dialog
```
`networkmanager` is the standard for controlling your network from the terminal.
`wpa_supplicant` adds support for WPA, WPA2, and WPA3 wireless networks.
`wireless_tools` allows a WiFi driver to expose it's functionality to userspace (Non-System) applications.
`netctl` is a CLI (Command Line Interface) frontend for `networkmanager`.
`dialog` is an application used by other programs to present dialog boxes within the terminal.

Additionally, we also want to make sure `networkmanager` is enabled:
```
systemctl enable NetworkManager
```

#### Enable LVM2 and Create the Initial Ramdisk

Next we want to install the `lvm2` package so that our new OS can read and write to our logical partitions:
```
pacman -S lvm2
```
We also want to enable the `lvm2` hook within our ramdisk environement. We're going to use `nano` to modify the config template (`/etc/mkinitcpio.conf`):
```
nano /etc/mkinitcpio.conf
```
This will open up a text editor within your terminal. You will want to navigate the cursor with the arrow keys and look for the last line starting with `HOOKS`. This should appear like:
```
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems fsck
```
It is import to note that this line should not have a `#` in front of it.

Once you have located this line, enter `lvm2` in between `block` and `filesystems`. As an example:

```
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block lvm2 filesystems fsck
```
If everything looks correct, press `Ctrl+X` to exit `nano`. You will be asked if you want to save the changes. Press `y` to save your changes. It will then ask for a file name, defaulting to the actual name of the file. You simple need to hit `enter` to continue and exit back to your shell.

Now that our config template is ready, we want to generate the actual config.

Do note that this only builds the initial ramdisk for the `linux` kernel. If you installed a different kernel (or multiple) you would need to run this command for each, replacing `linux` with your kernel(s):
```
mkinitcpio -p linux
```
`mkinitcpio` is a script used to create the initial ramkdisk, a very small environement which loads the specified kernel modules (such as `lvm2`) before passing control to `systemd`, our init system. The `-p` option just tells the script to use the default preset.

#### Generate the Locale

Next we want to generate locale information so our system can let software know what language to use. To start, we're going to use `nano` to modify `/etc/locale.gen`, a template file used to create the locale config:
```
nano /etc/locale.gen
```
You will want to locate your specific locale in this list. In my case, I'm a US English speaker, so I want to find `#en_US.UTF-8 UTF-8`. This would normally require scrolling down through the list, but we can search for our locale by pressing `Ctrl+W` and entering in `en_US`, then pressing enter. Once we find our locale, we want to remove the `#` "comment" that tells program to ignore that specific line. After this is done, the line should look like:
```
en_US.UTF-8 UTF-8
```
We can now exit `nano` with `Ctrl+X`, `y`, then hitting `enter`. Finally, we need to generate the locales.
```
locale-gen
```

#### Configure Users

Next we want to configure the user accounts. First, we want to change the password of the `root` user with `passwd`:
```
passwd
```
This will prompt you to enter a new password for the `root` account. You will not be able to see yourself typing, but you must type in the password twice.

Afterwords, we want to create our main user account (In the below example, the username I am using is `halornek`, replace this with your own username) with `useradd`:
```
useradd -m -g users -G wheel halornek
```
This will create the account `halornek`. Since we passed through the `-m` flag this will also create a folder under `/home` as `/home/halornek` for this users files. The first `-g` (lowercase) will define the primary group (`users`) that the account belongs to, while the second `-G` (Uppercase) defines any secondary groups. In this case, we have added the administrator group `wheel`.

Now we need to set a password for this user once again using the `passwd` command:
```
passwd halornek
```
You will once again be prompted to enter a password twice. It is generally advised that this password should be different than your `root` password, but this isn't entirely necessary. Just depends on how you want to set up your system.

#### Configure Sudoers

Within Linux, the concept of `sudo` is essentially `Run As Admin`. On most distrobutions this is done by adding users to either the `wheel` or `sudoers` group. In the case of Arch Linux, we normally use `wheel`. However, by default the `sudo` command does not allow `wheel` users.

To change this, we need to modify the `sudoers` file. It is not recommended to modify this file normally. Instead it should be modified with the `visudo` command. Do note that simply typing in `visudo` will use the default text editor `vi`, which many users will not be confortable with. Instead, we are going to force the editor `nano`:
```
EDITOR=nano visudo
```
This will open up the `sudoers` file within `nano`. We want to scroll down until we find a few lines that look like:
```
## Uncomment to allow members of group wheel to execute any command
#%wheel ALL=(ALL:ALL) ALL
```
We now want to remove the `#` comment from the second line so that it looks like:
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```
Once we are done we want to exit with `Ctrl+X`, `y`, then press `enter`.

#### Install the Bootloader

Finally, we need to install our bootloader, `grub`, along with a few additional packages necessary to set up your bootloader:
```
pacman -S grub dosfstools os-prober mtools efibootmgr
```
`grub` is the standard bootloader used by most Linux systems.
`dosfstools` allows the system to read and write to `FAT` filesystems.
`os-prober` is an optional application to scan for other operating systems that `grub` can hand off the boot process to.
`mtools` is an application to manage `DOS` entries. This doesn't really affect a `UEFI` boot, but might as well have it.
`efibootmgr` is similar to `mtools`, but for `UEFI` entries rather than `DOS`.

Before we can install our bootloader, we need to first mount the EFI partition we created at the start. We don't currently have the mountpoint created, so we need to create it with `mkdir`:
```
mkdir /boot/EFI
```
Once this is done, we can mount the EFI partition.
```
mount /dev/sdb1 /boot/EFI
```
Remember that the EFI partition is not part of our LVM volume, so we are mounting the first 500MB partition we created, `/dev/sdb1`.

Now we need to use `grub-install` in order to install our bootloader.
```
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```
This command tells `grub` to install agaist an `x86_64-efi` system using the bootloader ID `grub_uefi`, and then check that everything ran properly. You should receive a message saying `no errors reported`.

Next, we want to set the locale for `grub` so it uses the correct language. Do note that this command would change if you were using a different language:
```
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```
Finally, we want to generate the `grub` config file.
```
grub-mkconfig -o /boot/grub/grub.cfg
```
This will generate the config using the output file (`-o`) at `/boot/grub/grub.cfg`.

#### Exiting the Installer

At this point, we should be finished with the installation. You should now have a bootable version of Arch Linux. Let's go ahead and exit the `arch-chroot` environment:
```
exit
```
Next we want to unmount any extra drives (It's fine if there are a few errors):
```
umount -a
```
And finally, we want to restart our system:
```
reboot
```
With the EFI partition we installed earlier, it should be listed as your primary boot device and automatically boot into `grub`.

### Post Install

Now that we have a functioning system you can log into (Using the username and password created earlier) we can wrap up our install with features like a graphical environment and application "stores".

Before we will want to swap to the `root` user. This can be done using the `su` (Switch User) command:
```
su
```
This will require the password from your `root` account. If you want to enter the password from your user account use instead:
```
sudo su
```

#### Create the Swapfile

The swapfile (pagefile in Windows) is a file used to store data from RAM when your memory is getting close to full. This allows your system to free up RAM space as necessary by moving stale data to the storage drive. First, we will create the file using the `dd` utility.
```
dd if=/dev/zero of /swapfile bs=1M count=32768 status=progress
```
`if` is the "input file", or where `dd` is copying information from. In this case we told `dd` to use just a bunch of zeroes as the input file.
`of` is the "output file", or where `dd` is copying information to. In this case, it's a file named `/swapfile`.
`bs` is the block size to be written to. In our case, we're writing to a block size of 1MB.
`count` is the number of blocks that should be written. I normally like to make sure my swapfile is the same size as my RAM, in this case, 32GBs (Or 32 * 1024 = 32768).
`status` just specifies what information should be reported while the file is being created. In this case, we just tell `dd` to let us know the current progress.

Now that the swapfile is created we need to update the permissions of the swapfile with `chmod`:
```
chmod 600 /swapfile
```
`600` is the file permissions with the first number changing permissions for the user that owns the file (In this case, `root`), the group that owns the file (Also `root`), and the "world" (All users). `6` allows that user/group to read and write to the file, but not execute it. `0` allows no permissions. In this case, we are allowing the `root` user to read/write to the file while no one else can read/write and no one can execute it.
`/swapfile` is the target file's permissions we are modifying.

Next we need to actually define the file we've created as a swapfile. We can do this with `mkswap` (Make Swap):
```
mkswap /swapfile
```
Now we need to make sure the swapfile is mounted at boot. The easiest way to do this is to add it to the `fstab` file. Before we do so, we're going to make a backup of the `fstab` file in case something goes wrong using the `cp` (Copy Paste) command:
```
cp /etc/fstab /etc/fstab.bak
```
Now we can add the proper line using a combination of `echo` and `tee`:
```
echo '/swapfile none swap sw 0 0` | tee -a /etc/fstab
```
`echo` prints the specified text, in this case the line `/swapfile none swap sw 0 0`.
`|` is known as the "pipe" character. It can take the output from the left command (`/swapfile none swap sw 0 0`) and "pipe" it into the right command.
`tee` reads from an input (The out from before the `|`) and outputs it into the specified file. In this case, `/etc/fstab`.
`-a` tells `tee` to "append" to the output file, not overwrite. This will add the specified line to the end of the file.

Let's check what our new `fstab` file looks like:
```
cat /etc/fstab
```
It should look like:
```
# /dev/mapper/vga-lv_root
UUID=be8a58df-1ea1-47a9-8443-e0c2e62b4fdf       /               ext4            rw,relatime     0 1

# /dev/mapper/vga-lv_home
UUID=c11e490e-5f75-4fa1-9883-2b1a8947ee01       /home           ext4            rw,relatime     0 2

/swapfile none swap sw 0 0
```
If everything looks correct we can test the file by running:
```
mount -a
```
We can check if the swapfile is working by running:
```
free -m
```
To report the amount of free memory. This should look like:
```
               total        used        free      shared  buff/cache   available
Mem:           32031       15164       16859          38         501       16866
Swap:          32767       0           32767
```
Values may differ, but so long as `Swap:` is not just zeroes you are good.

#### Set Timezone

Next we want to define our timezone. I'm in US West so I'll be targeting Los Angeles as my time zone. You can list the exact timezones with:
```
timedatectl list-timezones
```
Once we locate our timezone (In my case, `America/Los_Angeles`) we can specify it:
```
timedatectl set-timezone America/Los_Angeles
```
Now we need to enable the time sync daemon.
```
systemctl enable systemd-timesyncd
```

#### Set the Hostname

Now we want to set the name of the system. In my case, I use `BD-DL` followed by an install number. This doesn't mean anything in particular, just what I use:
```
hostnamectl set-hostname BD-DL
```
Next we want to modify the `hosts` file so that local network applications can speak to each other.
```
nano /etc/hosts
```
The file should just look like:
```
# Static table lookup for hostnames.
# See hosts(5) for details.
```
We want to add a few hostnames to the end like so:
```
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1 localhost
127.0.1.1 BD-DL
```
Once we are done we want to exit with `Ctrl+X`, `y`, then press `enter`.

#### Additional Packages

Now we want to get started on installing additional packages. These are used in some way or other by the system.

##### Multilib

Before we move on we are going to enable the `multilib` repo, which is used to store 32-bit applications such as Steam. To do so, we need to modify our `pacman.conf`:
```
nano /etc/pacman.conf
```
Look for the lines near the bottom of the file that look like:
```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```
And remove the comments from them so they look like:
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```
Now we just need to refresh our repos.
```
pacman -Sy
```
`-S` syncs to the remote repos.
`y` requests a refresh of the repos.

##### Processor Microcode

First is your processor's microcode that allow the system to properly set performance levels on your CPU. This is going with the assumption you are on an AMD CPU. Replace the package with `intel-ucode` if using an Intel CPU:
```
pacman -S amd-ucode
```

##### Xorg

Next we want to install the `xorg` display server. `xorg` allows us to display a graphical environment. Do note that `wayland`, an upcoming display protocol is looking to replace `xorg`, but still does have some teething issues and may not function well depending on your graphics driver/display environement. If you are using an AMD GPU and either KDE Plasma or Gnome I would recommend looking into `wayland` at a later time.
```
pacman -S xorg-server
```

##### Graphics Drivers

There are really two options for GPU drivers in Linux. AMD and Intel cards use the open source `mesa` driver while Nvidia cards use the proprietary `nvidia` drivers. There is an open source Nvidia driver called `nouveau`, but this is really only used for displaying basic 2D environements.

###### AMD

```
pacman -S mesa vulkan-radeon lib32-vulkan-radeon
```

###### Intel
```
pacman -S mesa vulkan-intel lib32-vulkan-intel
```

###### Nvidia

```
pacman -S nvidia nvidia-utils
```

##### Flatpak

`flatpak` is a distro agnostic packaging format used by many distrobutions which allows developers to target a single environment. A lot of packages function great through `flatpak` and it is quickly becoming a unified standard.
```
pacman -S flatpak
```
Next we want to add the `flathub` report. `flathub` is the largest repository within `flatpak`. However, before we do this we want to exit the `root` account:
```
exit
```
Then run the command:
```
flatpak remote-add --if-not-exists --user flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```
After this we can go back to the `root` account.
```
su
```

##### KDE Plasma

Finally we are going to install our graphical environement. We have a lot of options for desktop environements and window managers within Linux, but for simplicity we're just going to go with KDE Plasma, a highly customizable and out of the box functional desktop environement that should be familiar enough to Windows users.
```
pacman -S plasma-meta kde-applications packagekit-qt5
```
`plasma-meta` is the package group for the KDE Plasma desktop.
`kde-applications` are basic programs and utilities for KDE Plasma.
`packagekit-qt5` is required for Plasma's `discover` app store to use `pacman` to install and update packages.

We also want to enable the login manager for KDE Plasma, `sddm`:
```
systemctl enable sddm
```

### Finishing Up

We should now be in a good spot to reboot straight into KDE Plasma.

All you need to do is enter:
```
reboot
```

Have fun.
