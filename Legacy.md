# windows-imaging 

### Legacy notes

This contains notes from older techniques that are no longer prioritized.  Hardware RAID setup has been supplanted by software RAID in the primary README file in this repo.  These may still be useful on older methods using Windows 7, and End-of-life hardware.

    Device Configuration – Verify config
        4. F12 on boot to enter Device Configuration
        5. Verify all Devices listed are: Healthy
        6. Navigate -> [left arrow key] to Intel Virtual RAID on CPU
        7. All Intel VMD Controllers > [ENTER]
            a. Note the listed drives:  If you see NVMe SSD, plan to put the OS on this drive.  There should be at least one additional drive.  Press ESC to return.
        8. Navigate -> [right arrow key] to AVAGO MegaRAID SAS Config Util
            a. Esc, Esc, Esc, YES (unless you make modifications)
        9. If they Array and disks look OK and do not need to be initialized, skip to 1.5

        1.4 Optional RAID: Notes for Custom and Other Models
            1.4.1 Initialize the primary RAID controller for C:\
Note: There are two possible options with the Older Precision towers, If the Integrated SAS RAID Controller was used and left enabled, skip to 1.5

    1.4.1.1 T7920 RAID Controller
    
    1. F12 on boot to enter Device Configuration
    2. Verify all Devices listed are: Healthy
    3. Navigate -> [right arrow key] to Intel Virtual RAID on CPU
    4. All Intel VMD Controllers > [ENTER]
    5. We want at least 1 virtual drive listed on this page.  Decide if you want to delete this VD and re-create, or if you have 0, initialize it from scratch
    6. If you want to delete the Array: Main Menu > Virtual Drive Management > Virtual Drive 0 RAID 0 > Operation > [Delete Virtual Drive], GO > Confirm  [Enabled] > YES, OK
    7. To create the Array: Main Menu > Configuration Management > Create Virtual Drive > Select Drives
    8. Enable Unconfigured Drives with your desired RAIDx level
    9. Save Configuration
    10. Confirm > YES > Enabled
    11. Operation > Fast Initialization > OK
    12. Virtual Drive Associated Drives > Enabled

---- 


    1.4.1.2 LSI RAID Controller (~T7600s)
    1a.  CTRL + R on Boot to configure the LSI RAID Controller
    2a.  Under primary LSI controller, Press F2, Clear Config
    3a.  Highlighting primary LSI controller, press F2, Create VD, RAID 10/6, select all disks
    4a.  Select Advanced, Check Fast Initialize, Name: Main, Click OK
    5a. If there are any unused drives in the array, they can be set as a hot spare in the event of a drive failure.  CTRL+N to move to drive list, Find any unused drive and highlight, F2, Make Global Hot Spare
    6a. Esc, OK, Reboot

-----

    1.4.1.3 Integrated RAID Controller  (~T7600s)
    1b. On boot, CTRL + C to configure the integrated RAID Controller
    2b.  Verify Boot Order 0
    3b. ENTER, Verify Boot Support = Enabled [BIOS & OS]
    4b. RAID Properties, Create RAID 1E/10 Volume, Create RAID 10 Volume, Save and Exit
    a. Drive Management > If you see [Drive Port 0, HDD, Online] skip to 1.4
            i. Note the drives available to use, if there are multiples, plan your RAID Config.  I recommend RAID6, or RAID10 When there are 4+ drives, or RAID1 Mirroring if there are two drives.  NEVER use RAID0 (unless you only run a single drive), and avoid RAID5 at all possible, unless you make your own backups extremely often.  RAID is never a substitute for backups, but with RAID0 and RAID5 you’re gambling greatly with losing your data.
            ii. Default configuration is 1 HDD, 1.636TB in Virtual Drive 0

Assume end users do not take proactive backup precautions,  plan for the safest route unless they specify otherwise.
