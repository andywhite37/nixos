# Dual booting NixOS and Windows 10 Pro

Below are the steps I took and various notes from my attempt to dual boot
NixOS with and existing Windows 10 installation on a new Lenovo Thinkpad X1
Carbon (6th Edition). I wanted to keep a small Windows partition for
gaming/etc. and install NixOS on a much larger partition for software
development. I'm by no means an expert with Linux system administration, and
I've never used NixOS before, so there are sure to be some speedbumps along
the way.

# Machine info

* Machine: Lenovo Thinkpad X1 Carbon (6th Generation)
* Ordered: November 2018
* Pre-installed OS: Windows 10 Professional
* Pre-configured BIOS version: 1.27
* SSD disk size: 1TB
* RAM: 16GB

# Installation steps & notes

## Preparing Windows and disk partitioning

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
