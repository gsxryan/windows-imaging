# windows-imaging
Bespoke Windows Imaging notes.  Interrupt the standard WinPE imaging sequence to support non-standard PC product models.  Inject drivers and other custom resources, so post imaging boot is successful.

This document describes how to use an SCCM Imaging CD-R (WinPE / PXE) to successfully deploy an image on unsupported hardware.  It outlines detail on how to setup RAID/Flash HBA mode, prepare BIOS & disk, apply the image, and perform additional top-off steps.

TLDR: The critical components for success in this method is to: 
1. Stop Imaging sequence before first automatic reboot by F8 command prompt in WinPE, open powershell to halt further progress.
2. diskpart, list disk, select disk X, clean, create partition primary, select partition 1, format fs=NTFS quick, active, assign
3. During the halt, and after placing the initial task sequence down, use DISM to inject custom drivers for first successful reboot after the standard image task sequences are complete.
4. dism /image:C:\ /Add-Driver /Driver:D:\M6700\Primary /Recurse /ForceUnsigned

This approach highly prioritized optimizing the long term T2 support burden, focusing on reducing need for on-site support response, keeping machines in patched compliance, utilizing existing imaging resources,  and has made choices to optimize these. 
- IT mode storage controllers (optional: software RAID only)
- AC power on after power cut
- VT/D IO: on for WSL/VMworkstation users
- Hibernation off (reduces complaints of HDD too full), For systems with 32GBor> RAM, pagefile is greatly reduced or OFF.

Windows 10 / 11 imaging, the PCs must have modern technology for this technique to work (UEFI boot, Safe boot, TPM).  For End-Of-Life hardware that does not have these capabilities, using Windows 7 image with this technique will still work.

### Storage controller - RAID vs HBA IT mode

For machines that have Storage controllers, at least from Dell, they usually come with hardware RAID mode enabled.  Business owners are highly unlikely to have the desire to reconfigure or recover a Hardware RAID controller, requiring an on-site technician or advanced remote KVM tools.  However, with direct attached storage drives, they're much more likely to swap a drive on their own without requesting service.  To lessen the long term burden of supporting this configuration for T2 support, I recommend switching to HBA IT mode.  IT mode directly passes through the hard drives without Hardware RAID configuration.  This allows easier setup, and recovery.  Users may still wish to use RAID, and in those cases they can setup software RAID.

-----

Applies to Dell Precision workstations: T79xx.  Most of these settings are already configured properly, but for best practice, verify them before you begin the image.  The most critical configuration items are (*).
M6700, M6800 note (or other single drive Non-RAID setup):  Be sure RAID On is selected SKIP to Section 1.2.

**1.1 Configure BIOS**
    
    1. F12 on boot to enter BIOS Setup
    2. General > 
        a. Boot Sequence: [UEFI] (*)
        b. Advanced Boot Options: [UNCHECK Legacy Option ROMs] (*)
        c. Date/Time: Verify this is correct CST Time
    3. System Configuration > 
        a. SATA Operation: RAID ON
            i. This item could differ depending on how you want to handle the onboard SATA ports.  This is not really applicable if you are using the PCIe card SAS RAID Controller.
        b. SMART Reporting: [CHECK] Enable SMART Reporting
    4. Security > Absolute > [disabled]
    5. Secure Boot > 
        a. Secure boot Enable: [SELECT] Enabled (*)
    6. Virtualization Support > 
        a. Virtualization: [CHECK] Enable Virtualization Technology
        b. VT for Direct I/O: [CHECK] Enable VT for Direct I/O
    7. Maintenance > 
        a. Asset Tag: Verify Hostname# is Set
        Apply, Exit and Reboot
-----
**1.2 Diagnostics**

    1. F12 on boot to enter Diagnostics
    2. Continue with Diagnostics, but do not allow extended memory tests.
    3. Exit (Esc) and reboot.

### Flash PERC card to HBA/IT Mode (”No RAID”)

This instruction is intended for use on the Avago MegaRAID card 9440-8i, but the same instructions should be useful if Dell changes the model for newer generations.  Or, if older machines come in for support.

If you have access to the Flash boot disk, skip to #3

    1) Create an EFI boot disk, with the IT/HBA mode BIN file ready to load
    See post 114 quote:
    Info on LSI SAS3408? Got myself a 530-8i... |
    
[ServeTheHome Forums](https://forums.servethehome.com/index.php?threads/info-on-lsi-sas3408-got-myself-a-530-8i-on-ebay.21588/page-6)

Creating the EFI Boot Disk for flashing PERC card to IT mode:
The primary tasks are as follows:

    Download Firmware & CLI
 
HBA 9400-8i Tri-Mode Storage Adapter (broadcom.com) (9400 operates in IT Mode only)
https://www.broadcom.com/products/storage/host-bus-adapters/sas-nvme-9400-8i

    • Click on 'Firmware' and get latest fimware file: 9400_8i_Pkg_P10_SAS_SATA_NVMe_FW_BIOS_UEFI.zip
    • Click on 'Management Software and Tools' and download latest STORCLI utility: STORCLI_SAS3.5_P10.zip
    2) Chose a Binary mode (EFI boot, or Windows x64 binary)
The following options will refer to A as EFI and B for Windows 

    Tested: Rufus EFI Freedos boot load
    
BUILD - [HOW-TO] Flashing LSI SAS HBA Controller [EFI/UEFI] | TrueNAS Community
https://www.truenas.com/community/threads/how-to-flashing-lsi-sas-hba-controller-efi-uefi.78457/

    After building EFI drive
    Copy below files to USB drive:
     • ..\EFI\storcli.efi
     • ..\Firmware\HBA_9400-8i_SAS_SATA_Profile.bin
    2a) Optional: Backup the firmware.  (this didn’t work in EFI or windows for me yet) Backup the SAS Config
    storclix64.exe /c0 show all
    storcli /c0 show all >> Hostname.txt

    • Backup misc flash regions to files. Just in case you need them later.
        #storcli /c0 get bios file=backup_bios
        #storcli /c0 get firmware file=backup_firmware
        #storcli /c0 get mpb file=backup_mpb
        #storcli /c0 get fwbackup file=backup_fwbackup
        #storcli /c0 get nvdata file=backup_nvdata
        #storcli /c0 get flash file=backup_flash


3) Disconnect all local drives on the machine temporarily.  Don’t forget the NVME PCIe card
This protects the drives from the boot disk, and makes it easier to locate the USB mount in EFI boot.
Disconnect the Ethernet plug temporarily.
Place a 2pin Jumper on PIN J4 on the PERC Card	.  You’ll be able to read the PIN on the breadboard.
 (do this with machine off)
Reinstall the PERC card.

**Boot to the EFI USB environment with the boot disk.**

    A) Boot to UEFI, UEFI bios must be on
    In EFI prompt type map
    Chose the USB Drive ~”fs0”
    Type mount fs0
    Type fs0:
    Type DIR to verify you’re on the USB

    6) flash firmware
    • A) Flash firmware:
    storcli.efi /c0 download file=HBA_9400-8i_SAS_SATA_Profile.bin
    once complete it will show: Description = CRITICAL! Flash successful. Please power cycle the system for the changes to take effect
    • Type “exit”, Power off.
    • Remove jumper from J4 header.
    • Reinstall PERC Card into original PC
    • Unplug USB Boot Drive (EFI boot)
    • Reconnect Ethernet cord
    • Reconnect All Hard Drives, Except if you are about to image.  Leave the front bays unplugged if you want to put the OS on the NVME PCIe Card.  Once imaged you can plug the drives back in.

Options in Legacy.md indicate usage with MegaRAID Hardware RAID Mode instead of HBA/IT Mode, please make sure you really want to do that before using those instructions.  That method most likely requires on-site responses for failed arrays or initializations.  The mode above likely will help reduce on-site work.

**Download or acquire the Boot Media CD**

    1. Copy the boot media locally and burn a CD, or acquire a USB Boot Drive, or boot from PXE
    On Boot, F12, UEFI: Dell DVD+/-RW DW (Don’t step away yet!)
    Press any key to boot… [Press a single key once]
    2. Once the “Welcome to the Task Sequence” appears, Press F8
    3. Type: ipconfig, to Verify a valid IP address has been given.  If you don’t have an address, this address needs to have NAC/802.11q disabled, and added inside SCCM imaging boundaries.  If you were disconnected, Plug an ethernet cable in and type ‘ipconfig’ in again.
    4. initialize the drive using diskpart
    5. type: diskpart, list disk, select disk 0, clean
        a. Note after running commands above: Verify that Disk 0 is the Drive you wish to install the OS on.  If not, you need to temporarily disconnect a RAID controller, or reconfigure the array
        b. Also list vol to verify the C: drive letter is not used.  If C:\ letter is already used, type: select vol # (vol letter with C:\), assign letter=R (or any other unused driveletter)
    6. For each comma, hit <ENTER> Type: select disk 0, clean, convert gpt, create partition primary,       select partition 1, format fs=NTFS quick, active, assign letter=C, rescan 
    7. Type: list vol (This will allow you to reverify that C:\ is unused)
    8. Type: exit, exit (to close cmd window)


    1.6.2 Scan for Image Task Sequence
    9. Click Next>, wait for task sequence to appear.

Task Sequence Splash Screen


    1.6.3 Begin to pull the Task Sequence 
    1. Select the correct Task Sequence.
        a. Highlight the standard imaging task sequence.


    Image Task Sequence Selection
        2. Click Next and the task sequence begins.  FAILED TO RUN?: See Section 1.9

Task Sequence Progress Bar


    1.6.4 Enter Computer Information
        Data Confirmation
    Fill in the required information.  Click Authenticate > Start Image

**1. Let the image continue.  F8 and type ‘powershell’ (this prevents reboot).  Keep PowerShell open until the installation progress bar disappears.  If it rebooted, DO NOT let it start back up automatically (boot to CD and go back into powershell to continue).  Continue to next section to pull drivers manually.**
 
WIM Application Progress Bar

        1.7 Driver Injection
            1.7.1 Create the custom machine Drivers CD
    1.  Burn a new CD With the custom machine Drivers, or copy them to USB.  
To create your own, go to support.dell.com and instead of installing the drivers, extract the packages to the external media.

            1.7.2 Inject Custom Drivers to C:\
    If you rebooted, boot immediately back into the imaging CD.  If you get a Blue-Screen (0x7B), or Fail Imaging 0x01 it will require that you start the imaging task sequence over again. This is due to RAID or NIC drivers missing on the C:\ drive installation.   These instructions will manually load unsupported drivers onto the machine to continue the “preparing windows for first use” portion of the imaging steps.
    1. Boot to the Imaging CD, (or if you are already there continue)
    2. F8, but do not image this time. (diskpart, list vol (find driveletter for CD))
    3. Insert the media that contains the custom machine Drivers 
    4. Type DISM command below (forceUnsigned may be advised to omit this, as you want to make sure the Dell drivers you acquired are verified): 
```
DISM  /image:C:\   /Add-Driver  /Driver:D:\T7920   /Recurse   /ForceUnsigned
```
		Occasionally the CD Drive volume will change, to check which driveletter do ‘diskpart’ “list volume”
	4.  Verify drivers have installed successfully
	5.  Exit, Reboot, Remove the CD, The image should continue, and succeed.

NOTE: if the reboot kicks off a hardware diagnosing error, power off the system manually, and turn back on.  The hardware error should not reappear.


        1.8 Post-Image Installation Steps
    When your Windows 10 installation for a laptop is complete, OSD presents a completion screen after the final reboot. 
            1.8.1 Complete OSD
    1. Click, or TAB over to “Continue to Windows” on the Deployment Complete screen.
Windows 10 Completed Deployment

            1.8.2 Install Missing Drivers
    Performing the DISM steps above usually installs all required drivers, but a quick check here will make sure every device is operating.

    [OPTIONAL] Start up the PC, login, and install non-standard drivers
Recommended: The 7x20T*.exe (BIOS upgrade)
You can alternatively download the latest from support.dell.com
Once you’ve installed all the drives, you should see a clean Device Manager.

            1.8.3 Restrict the Pagefile & Hibernation file to conserve disk space (optional).

If you have a massive storage array on the OS, or the user has upgraded the RAM, a user may complain that the Hard drive is excessively full.
1. Run “powercfg -h off”
2. Adjust PageFile OR if the machine has lot of RAM – DISABLE it.  (Right Click Computer> Properties > Advanced System Settings > Advanced Tab > Performance: Settings button > Advanced tab > Change Button > Custom Size > Intial: 10000, Maximum 50000
3. Disable Write caching on the SSD Drive for hard/power cut protection.
Device Manager – Disk Drives – M.2 - Right click, properties – Policies Tab, UNCHECK all boxes.


        1.9  Failure notes

    1. Should the image join the domain, but fail installing some later items, it may be OK to continue, and harden with the remaining steps manually.   The smsts.log can be investigated to determine which step failed.  Continue where it left off to complete the hardening steps.  This was a common occurrence with Previous models on Win7 but doesn’t seem to happen as often with Win10 and 7920s.
