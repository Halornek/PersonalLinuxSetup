# PersonalLinuxSetup

This document is primarily designed as a failsafe to restore my Linux Desktop OS setup in case I ever reach an unrecovable state.

## System Specifications:
```
CPU: Ryzen 9 5950X
Motherboard: Gigabyte X570 Aorus Master
RAM: 2 x 16GB Trident Z Neo DDR4-3600 @CL16
GPU 1: Gigabyte Reference Radeon RX 6900 XT
GPU 2: EVGA Geforge RTX 3070 XC3 (FHR)
SSD (Windows): Sabrent Rocket NVMe4 1TB
SSD (Linux): Crucial P5 NVMe3 1TB
SSD (Shared): Crucial BX500 2.5" SATA III
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
Once diskpart has loaded, simply type:
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
If your primary boot disk has a * under Gpt, then Windows is already installed under UEFI boot.

If there is not a * under Gpt for your primary boot disk, then Windows is running under legacy (MBR) boot. You can convert it using the Windows tool [MBR2GPT](https://docs.microsoft.com/en-us/windows/deployment/mbr-to-gpt).

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
Right click the directory "TimeZoneInformation", go to "New", and select "DWORD (32-Bit) Value".

Name the value "RealTimeIsUniversal", set the value to 1, and ensure the base is set to Hexadecimal.

#### Disable Fast Startup

*Can be ignored if using VM*

Windows default behavior when instructed to "Shut Down" is to instead put the drive into a deep sleep state. This makes the drive write protected and would prevent your Linux OS from writing anything to the drive if you used "Shut Down" in Windows instead of "Restart".

This can be avoided by opening Command Prompt/Powershell as admin (Shift right click on the start menu) and entering:
```
powercfg -h off
```

## Install Ubuntu

This document is assuming that the Linux host OS will be running some variant of Ubuntu 20.04 LTS. At time of writing, current standard is 20.04.4 LTS.

My current recommendation is [Kubuntu](https://kubuntu.org/) due to my preference of KDE Plasma instead of Gnome.

Follow the standard installation process to your prefences. If you decide to dual boot you may need to change a few settings.]

It is recommended to install 3rd party drivers.

## Update and Utilities

Run update and upgrade.
```
sudo apt update
```
```
sudo apt upgrade
```

#### Install Browser

You can install most browsers through the Discover app fairly easily. My current preference is Edge, which does require a bit more legwork.

Download and save the .deb package from Microsoft's [Evergreen](https://www.microsoft.com/en-us/edge?r=1#evergreen) branch.

Navigate to your downloads folder and run the .deb package with package manager.

Run updates:
```
sudo apt update
```
If an error is received about an unsigned key, review the link [here](https://askubuntu.com/questions/13065/how-do-i-fix-the-gpg-error-no-pubkey) to add the key.

Open "Default Applications" and set Edge to the default browser.

#### Install OpenSSH Server

I like to be able to SSH into my system to troubleshoot issues with virtualization. This is especially helpful when trying to perform single GPU passthrough.

I accomplish this with [OpenSSH](https://www.openssh.com/) Server.
```
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

#### Install NeoFetch

Because every Linux user needs to take pictures of their desktop with [NeoFetch](https://github.com/dylanaraps/neofetch) running.
```
sudo apt install neofetch
```

#### Install bpytop

My preferred top alternative is [bpytop](https://github.com/aristocratos/bpytop)
```
sudo snap install bpytop
```

#### Install tmux

I consider [tmux](https://github.com/tmux/tmux/wiki) mandatory, not just for command line multiplexing but also the ability to run a terminal shell in the background and be able to re-open it via SSH.
```
sudo apt install tmux
```

#### Install Discord

My general go-to chat platform is [Discord](https://discord.com/).
```
sudo snap install discord
```

#### Install Git

Not really much to say here. [Git](https://git-scm.com/) is all but mandatory for things like kernal compilation and software version control.
```
sudo apt install git
```

#### Install Tree

[Tree](http://mama.indstate.edu/users/ice/tree/) is a useful application for quickly visualizing a file tree within a terminal session.

## Install OpenRGB

My system utilizes RAM and motherboard RGB components that I can't apply hardware profiles to inside of Windows. To gain control of these components I like to utlize [OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB).

#### Install Build Dependencies

I found the following dependencies necessary while compiling and setting up OpenRGB.
```
sudo apt install git build-essential qtcreator qt5-default libusb-1.0-0-dev libhidapi-dev pkgconf libmbedtls-dev
```

#### Clone From Source

Now that we have our dependencies we will want to clone the source repo.
```
git clone https://gitlab.com/CalcProgrammer1/OpenRGB
```
Navigate into the new directory.
```
cd OpenRGB
```
Run qmake.
```
qmake OpenRGB.pro
```
Run make (Replace 32 with number of CPU threads).
```
make -j32
```
Update the path for a terminal shortcut.
```
sudo ln -s ~/OpenRGB/openrgb /usr/local/bin/openrgb
```

## Patch Kernel for OpenRGB

OpenRGB requires a customized Kernel with L2C bus control enabled in order to access DRAM RGB configurations. Full instructions can be found [here](https://gitlab.com/CalcProgrammer1/OpenRGB/-/wikis/OpenRGB-Kernel-Patch).

#### Install Build Dependencies

Before taking any further steps ensure that you have returned to your home directory in Terminal.
```
cd ~/
```
I found the following dependencies necessary when attempting to compile the modified kernel.
```
sudo apt install flex bison libssl-dev libc6-dev libncurses-dev libelf-dev
```

#### Clone and Patch Kernel

We now want to clone the source [Linux](https://github.com/torvalds/linux) kernel tree.
```
git clone https://github.com/torvalds/linux
```
Navigate into the new directory.
```
cd linux
```
Checkout a valid version. I found the best results using 5.11 for AMDGPU compatibility.
```
git checkout v5.11
```
Apply the patch from OpenRGB.
```
patch -p1 < ~/OpenRGB/OpenRGB.patch
```
Copy the existing configuration file. This may differ depending on when you installed Kubuntu.
```
cp /boot/config-5.11.0-44-generic .config
```
Verify signing keys to prevent certificate error and disable Info BTF.
```
nano .config
```
Press Ctrl+W to search for "CONFIG_MODULE_SIG_KEY" or "CONFIG_SYSTEM_TRUSTED_KEYS". Comment out (Add "# " before) the lines for "CONFIG_MODULE_SIG_KEY" and "CONFIG_SYSTEM_TRUSTED_KEYS".

Press Ctrl+W to search for "CONFIG_DEBUG_INFO_BTF". Use arrow keys to navigate to the value and if set to "y", change to "n".

Make oldconfig. May prompt for new config settings. Just hold enter until complete.
```
make oldconfig
```
Configure menuconfig.
```
make menuconfig
```
Navigate to Device Drivers > I2C Support > I2C Hardware Bus Support > Nuvoton NCT6775 and compatible SMBus controller. Once highlited hit "Y" to mark as a build module.

Exit and save configuration.

#### Build .deb Packages

Run make (Replace 32 with number of CPU threads).
```
make -j32
```
You may have to hit enter a few times at the start to apply default settings for "CONFIG_MODULE_SIG_KEY" and "CONFIG_SYSTEM_TRUSTED_KEYS".

#### Install .deb Packages

First navigate back to your home directory.
```
cd ..
```
Run dpkg.
```
sudo dpkg -i *5.11.0+-1_amd64.deb
```

#### Verify

Reboot the system.
```
sudo reboot
```
After the reboot run uname to verify current kernel.
```
uname -a
```
You should see "5.11.0+"
```
Linux bd-dl-001 5.11.0+
```
If the new Kernel is not running, us GRUB to manually select the new Kernel under "Advanced Options for Ubuntu".

Should GRUB not automatically prompt you by default, press ESC (If UEFI boot) to force GRUB to launch. This may take some trial and error to get used to the timing as it needs to be after UEFI initialization but before Linux has been booted.

## Grant SMBuss Access

TBC. . .
