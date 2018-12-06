# Dual booting NixOS and Windows 10 Pro

Below are the steps I took and various notes from my attempt to dual boot
NixOS with and existing Windows 10 installation on a new Lenovo Thinkpad X1
Carbon (6th Edition). I wanted to keep a small Windows partition for
gaming/etc. and install NixOS on a much larger partition for software
development. I'm by no means an expert with Linux system administration, and
I've never used NixOS before, so there are sure to be some speedbumps along
the way.

My general strategy is to try to keep the pre-existing Windows 10
install intact, shrink the Windows OS partition down to 200GB, and then
create partitions for the NixOS installation and Linux swap in the
remaining space after shrinking the Windows partition.  This will all be
done using Windows-based disk partitioning tools.

Then, during the NixOS install, I will assume that the pre-created EFI
boot partition will work with the Linux boot loader stuff, and leave it
untouched, and only format the newly-created nixos and swap partitions,
and mount them as directed in the NixOS installation guide.

Update: in the end, there was quite a bit of reading and crossing of
fingers involved, but I got NixOS installed and up and running with no
problems at all!  Also, at no point in the process was I unable to boot
into the Windows OS!  Leaving all the existing Windows-based partitions
alone (other than shrinking the Windows partition), and using the
Windows tools to create partitions, then formatting these partitions
with the Linux tools appears to work fine.

# Machine info

* Machine: Lenovo Thinkpad X1 Carbon (6th Generation)
* Ordered: November 2018
* Pre-installed OS: Windows 10 Professional
* Pre-configured BIOS version: 1.27
* SSD disk size: 1TB
* RAM: 16GB

# Preparing Windows and disk partitioning

1. Boot into Windows
1. Get Windows up-to-date
    * Install all Windows updates (using Windows `Check for Updates` tool)
    * Install all Lenovo updates (using Lenovo `System Updates` tool
      (external download from Lenovo)
        * https://support.lenovo.com/us/en/solutions/tvsu-update
        * This included an update to update the BIOS from 1.27 to 1.34
        * I understand there might have been some issues with Linux Suspend on this machine that were fixed in BIOS version 1.30, so that is the main motivation I have for doing this.
1. Run `msinfo32.exe` and note information about the BIOS:
    * BIOS Version/Date: `LENOVO N23ET59W (1.34), 11/8/2018`
    * BIOS Mode: `UEFI`
1. Run `diskmgmt.msc` to see disk partitions (according to Windows
   built-in tool):
    * The tool shows 3 initial partitions:

        |Volume|Layout|Type|File System|Status|Capacity|Free Space|% Free|
        |-|-|-|-|-|-|-|-|
        |(Disk 0 partition 1)|Simple|Basic||Healthy (EFI, System Partition)|260MB|260MB|100%|
        |Windows (C:)|Simple|Basic|NTFS (BitLocker Encrypted)|Healthy (Boot, Page File, Crash Dump, Primary Partition)|952.62GB|910.15GB|96%|
        |WinRE_DRV|Simple|Basic|NTFS|Healthy (OEM Partition)|1000MB|571MB|57%|

    * Note: I had installed a couple things in Windows prior to this, using some of the c: disk space.
1. At this point, I thought I would try a few 3rd-party disk partition tools to see if they might make the task of shrinking the c: parition to make room for a Linux install any easier.
    * `Paragon Partition Manager` (free edition)
        * Seemed okay, but free version felt very limited
    * `MiniTool Partition Wizard` (free edition)
        * Same observation as Paragon
    * `Macrorit Partition Expert` (free edition)
        * At first glance, I liked this tool the best out of these, so I will try to use this one if I need it.
1. Run `Macrorit Partition Expert` tool
    * This tool reported the following partition info:
        * Disk 0: Basic GPT, Healthy

            |Volume|Capacity|Free Space|File System|Type|Status|
            |-|-|-|-|-|-|
            |*: SYSTEM|260.0 MB|230.9 MB|FAT32|GPT(EFI System Partition)|Healthy(System)|
            |*:|16.00 MB|16.00 MB|Not Formatted|GPT(Reserved Partition)|Healthy|
            |C: Windows|952.6 GB|9210.2 GB|BitLocker Encrypted|GPT(Data Partition)|Healthy(Boot)|
            |*: WinRE_DRV|1000.0 MB|571.0 MB|NTFS|BPT(Recovery Partition)|Healthy|

    * So apparently there is a sneaky little reserved partition that was hidden from the built-in Windows Disk Management tool.
1. The next step is to shrink the `C:` partition to 200GB (that's the size I'm choosing) to make room for a large partition for NixOS, and maybe a swap partition for Linux.  I'm going to go with the 2xRAM strategy for swap, and shoot for a 32GB swap partition.
    * I'm referencing the following articles for what to do here:
        * https://zimbatm.com/journal/2016/09/09/nixos-window-dual-boot/
        * https://www.download3k.com/articles/How-to-shrink-a-disk-volume-beyond-the-point-where-any-unmovable-files-are-located-00432
    * However, I'm going to attempt to use the `Macrorit` tool to hopefully avoid the extra steps of getting rid of the unmovable pagefile.sys, etc. files to allow the partition to be shrunk more.  Not sure if it works that way...
    * `Macrorit` will not let me shrink the `C:` parition now, because it is Bitlocker Encrypted
    * I'm not sure whether I will need to mess with the unmovable Windows files when using `Macrorit` or if it somehow takes care of that for you.
1. Decrypt the `C:` partition to allow for shrinking
    * See: https://macrorit.com/help/resize-bitlocker-encrypted-partition.html
    * Basicall run `Manage BitLocker` from Windows, and click `Turn off BitLocker`
    * This took about 2-3 minutes
1. Run `Macrorit` again (or click `Reload Disks`)
    1. note that `C:` now has File System `NTFS` (no longer Bitlocker Encrypted), and the menu option to resize the parition is available now.
    1. Right-click `C: Windows` and select `Resize/Move Volume`
    1. I'm going to attempt to resize to `200,000 MB` (`200 GB`), which will leave `757.3 GB` unallocated.
    1. This enqueues an action to resize - so click `Commit` to apply the change
    1. `Macrorit` immediately restarts the computer
    1. A black screen appears, and indicates that the volume was successfully reszied
    1. Windows restarts
    1. Login to Windows, open Explorer, and note that the `C:` drive now has capacity of about `195 GB`
    1. So in the end, it *appears* that `Macrorit` takes care of the pagefile.sys/hiberfil.sys/etc. unmovable file stuff for you when resizing the volume...
1. At this point I could re-enable BitLocker on the Windows partition, but I don't really care, so maybe I'll just do that later (or maybe not).
1. The `zimbatm.com` guide above now recommends to create a new `NTFS` partition in Windows Disk Management for the NixOS install.  I'm going to create two partitions: a 32GB partition for Linux swap and another partition using up the rest of the space for NixOS.
    1. For the unallocated space `Disk Management` shows maximum disk space as `775,484 MB`, so I will make the NixOS partition `743,484 MB` and the other `32,000 MB`
    1. Select `Do not assign a drive letter or drive path`
    1. Select `Format this volume with the following settings`
        * Note: I doubt any of this really matters, as we will likely be re-formatting during the NixOS install
        1. File system: `NTFS`
        1. Allocation unit size: `Default`
        1. Volume label: `nixos` (or `linux-swap` for the second partition)
            * I'm just labelling these here so they might be easier to recognize in the Linux partition tools.
            * These labels will get blown away and reset when we format the disks later.
        1. Perform a quick format: Checked
        1. Enable file and folder compression: Unchecked
    * Now `diskmgmt.msc` shows the following partitions (in the order on the disk diagram):

        |Volume|Layout|Type|File System|Status|Capacity|Free Space|% Free|
        |-|-|-|-|-|-|-|-|
        |(Disk 0 partition 1)|Simple|Basic||Healthy (EFI, System Partition)|260MB|260MB|100%|
        |Windows (C:)|Simple|Basic|NTFS (BitLocker Encrypted)|Healthy (Boot, Page File, Crash Dump, Primary Partition)|952.62GB|910.15GB|96%|
        |nixos|Simple|Basic|NTFS|Healthy (Primary Partition)|726.06GB|725.88GB|100%|
        |linux-swap|Simple|Basic|NTFS|Healthy (Primary Partition)|31.25GB|31.17GB|100%|
        |WinRE_DRV|Simple|Basic|NTFS|Healthy (OEM Partition)|1000MB|571MB|57%|

    * And just for the heck of it `Macrorit` shows the following:

        |Volume|Capacity|Free Space|File System|Type|Status|
        |-|-|-|-|-|-|
        |*: SYSTEM|260.0 MB|230.9 MB|FAT32|GPT(EFI System Partition)|Healthy(System)|
        |*:|16.00 MB|16.00 MB|Not Formatted|GPT(Reserved Partition)|Healthy|
        |C: Windows|952.6 GB|9210.2 GB|BitLocker Encrypted|GPT(Data Partition)|Healthy(Boot)|
        |*: nixos|726.1 MB|725.9 MB|NTFS|GPT(Data Partition)|Healthy|
        |*: linux-swap|31.25 MB|31.17 MB|NTFS|GPT(Data Partition)|Healthy|
        |*: WinRE_DRV|1000.0 MB|571.0 MB|NTFS|BPT(Recovery Partition)|Healthy|
1. Next, I read through the [Arch Linux dual booting guide](https://wiki.archlinux.org/index.php/Dual_boot_with_Windows), and it recommended to disable `Fast startup` in Windows to avoid some issues.
    * This links to this page for Windows 10: https://www.tenforums.com/tutorials/4189-turn-off-fast-startup-windows-10-a.html
    * Basically go to `Control Panel -> System & Security -> Power Options` and click `Choose what the power buttons do` -> `Change settings that are currently unavailable` -> Uncheck `Turn on fast startup (recommended)` -> `Save changes`
1. Finally, we need to disable `Secure Boot` in the BIOS, so we can boot from the NixOS install USB.
    1. Restart Windows, and when the Lenovo screen comes up, hit `Enter` to interrupt startup, and `Enter` again to pause the countdown timer.
    1. Hit `F1` to enter BIOS Setup Utility
    1. Go to the `Security` tab, and go down to `Secure Boot` and hit `Enter` to open the config screen.
    1. With `Secure Boot` `Enabled` selected, hit `+` to toggle it to `Disabled`
    1. Hit `F10` to save & exit

# NixOS install

The version of NixOS that I will be using is 18.09, with the graphical installer.

1. Download the NixOS installation .iso and write it to a USB drive
    1. Instructions: https://nixos.org/nixos/manual/index.html#sec-booting-from-usb
1. Insert the USB stick in the machine and start/restart Windows, so we can boot from the USB drive.
1. At the `Lenovo` screen hit `Enter` to interrupt startup and `Enter` again to pause
1. Hit `F12` to choose a temporary startup device
1. For me, I select `USB HDD: USB1203 Flash Disk`
1. The NixOS screen comes up, and I selected the first option `Installer`
1. Some things happen, then I end up on the screen `<<< Welcome to NixOS 18.09.1534.d45a0d7a4f5 (x86_64) - tty1 >>>`
1. Run `systemctl start display-manager` so we can see what graphical tools are provided
1. KDE Plasma starts up with `GParted`, `Konsole` and `NixOS Manual` icons, and other stuff in the main menu in the bottom left
1. Now click the networking icon in the lower-right and connect to a WiFi network to make sure networking is setup and working
    * worked without a problem for me
1. Side note: for the entire install, I'm going to follow the `UEFI/GPT` steps, rather than the legacy `BIOS/MBR` steps.
    * I know very little about all this stuff, but `UEFI` seems to be the right answer.
1. I'm going to skip the `partition` section of the NixOS install guide, since I already created the partitions using the Windows tools (hopefully...)
1. Now I need to format the `nixos` and `linux-swap` partitions for the Nix install and swap.
    1. Based on the `zimbatm.com` guide, I'm assuming we can use the existing EFI partition that came with Windows and not mess with that (i.e. not format it).
    2. I'm going to use the graphical `GParted` tool rather than the CLI tools for this, to hopefully reduce the chance of doing something unwanted via CLI tools that I'm not super familiar with.
    1. Run `GParted` (graphical `parted` interface)
    2. `GParted` shows the following partitions:

        |Partition|Name|File System|Label|Size|Used|Unused|Flags|
        |-|-|-|-|-|-|-|-|
        |/dev/nvme0n1p1|EFI system partition|fat32|SYSTEM|260.00 MiB|33.08 MiB|226.92 MiB|boot, hidden, esp|
        |/dev/nvme0n1p2|Microsoft reserved partition|unknown||16.00 MiB|--|--|msftres|
        |/dev/nvme0n1p3|Basic data partition|ntfs|Windows|195.31 GiB|40.69 GiB|154.63 GiB|msftdata|
        |/dev/nvme0n1p4|Basic data partition|ntfs|nixos|726.06 GiB|180.50 MiB|725.88 GiB|msftdata|
        |/dev/nvme0n1p5|Basic data partition|ntfs|linux-swap|31.25 GiB|75.29 MiB|31.18 GiB|msftdata|
        |unallocated||unallocated||1.00 MiB|--|--||
        |/dev/nvme0n1p6|Basic data partition|ntfs|WinRE_DRV|1000.00 MiB|424.98 MiB|575.02 MiB|hidden, diag|

    3. I'm going to skip the disk encryption stuff for now, and just go for unencrypted ext4... hopefully I won't regret that later.
        * The `zimbatm.com` guide shows how to do disk encryption with `ext4` for reference
    4. Select the `nixos` partition, and `Partition -> Format to -> ext4`
        * Tempting to try `btrfs` here, but I'll stick to the guide's suggested `ext4`
    5. Select the `linux-swap` partition and `Partition -> Format to -> linux-swap`
    6. `GParted` shows 2 operations pending:
        1. `Format /dev/nvme0n1p4 as ext4`
        2. `Format /dev/nvme0n1p5 as linux-swap`
    7.  This seems right, so click `Apply`
    8.  Now label the `nvme0n1p4` partition as `nixos` and the `nvme0n1p5` partition as `swap` and click apply
        1. `GParted` won't let me label the pre-existing `nvme0n1p1` (EFI) partition as `boot`, probably b/c I didn't create it.
1. Now mount the nixos partition and the boot partition.
    * This differs a little from the `zimbatm.com` guide - he mentions wondering why he didn't get the `/dev` devices with `nvme...` names, but I seem to have gotten them.
    * Also note that we're not using the `/dev/sd*` names as mentioned in the NixOS setup guide - I'm using the `/dev/nvme...` names.  Hopefully this works.
    1. `mount /dev/nvme0n1p4 /mnt`
    2. `mkdir /mnt/boot`
    3. `mount /dev/nvme0n1p1 /mnt/boot`
1. Enable swap on the swap partition
    1. `swapon /dev/nvme0n1p5`
1. Create the nix config file
    1. `nixos-generate-config --root /mnt`
1. Edit the `/mnt/etc/nixos/configuration.nix` file
    * Referencing example here:  https://github.com/fooblahblah/nixos
    * Use `networking.networkmanager.enable = true` rather than `wireless` (not sure why, but the `zimbatm.com` guide says to).
    * Note: In the NixOS dual boot guide, it mentions using `grub` settings for OS discovery but this was not mentioned in the `zimbatm.com` guide (presumably because we didn't create the EFI partition, so we don't need grub to get involved there?), so I'll skip it for now and see what happens.
    * TODO: once this is settled post the annotated file I ended up with
1. Run `nixos-install` to build the config and install all the pkgs
    * Note: if you have bad configs, it will stop and tell you what's wrong.
    * Once it finally builds and installs, you get prompted to enter the `root` password
1. Run `reboot`
1. Hopefully the system reboots, and you are presented with an OS selection screen!
    * This means that you indeed do not need to setup the `grub` boot loader stuff for OS discovery.  I don't know enough about any of this to say more.
    * Note that you don't need to hit `Enter` to interrupt the boot process on the `Lenovo` screen, just let it go and it should stop on the OS selection screen.
    1. To run NixOS select `NixOS`
    2. To run the pre-existing Windows 10 OS select `Windows Boot Manager`
1.  In my nix config, I specified to create a user `awhite`, but I couldn't login with an empty password, so I logged in as `root` and ran:
    1. `sudo passwd awhite` and entered a new password
    2. Now I can login as `awhite`
    3. You may need to re-configure WiFi as your new user
1. At this point, everything seems to be working, but I'm sure there will be lots of tweaking going forward, and lots more research needed.
