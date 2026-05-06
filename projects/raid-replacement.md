---
layout: page
title: "Failed RAID Drive Replacement & Symbolic Link Setup Project"
permalink: /projects/raid-replacement/
---

# Failed RAID Drive Replacement & Symbolic Link Setup Project

## Overview
A dental practice client (Anon Dental) was operating on aging SATA storage that needed to be migrated to a new RAID 5 array of SAS drives on their Dell PowerEdge T340 server. The challenge was migrating roughly [X TB] of imaging data without changing the drive path the Gendex imaging software was tied to, and without disrupting the staff's workflow at any of the workstations.

The solution combined a Dell PERC H730P RAID rebuild with a Windows directory junction (`mklink /J`) so the application kept writing to the same `E:\Gendex Images` path while the data physically lived on the new array.

## Environment

![Server closet overview showing rack, tower servers, and cabling](/assets/projects/raid-replacement/00-server-room.jpg)

- Dell PowerEdge T340 server, Windows Server [version]
- Dell PERC H730P Adapter RAID controller
- Existing storage: SATA-based RAID 5 virtual disk hosting `E:\Gendex Images`
- New hardware: 4 x 1.2TB Dell SAS drives (added to empty bays)
- Application: Gendex imaging software reading and writing to `E:\Gendex Images`
- Share: SMB share of `E:\Gendex Images` mapped from staff workstations via Group Policy drive maps

![New 1.2TB Dell SAS drive in caddy](/assets/projects/raid-replacement/01-new-drive.jpg)

## Goals
- Migrate imaging data to faster, more reliable SAS storage
- Configure the new array as RAID 5 for single-disk redundancy with maximum usable capacity
- Preserve the existing UNC path and drive letter so no workstation reconfiguration was needed
- Limit business-hours impact to zero, with the cutover happening after close

## Approach
1. **Phase 1 (business hours):** install new drives, create the new RAID 5 virtual disk, initialize the volume in Windows
2. **Phase 2 (business hours):** pre-stage a full mirror of imaging data to the new volume with FreeFileSync, while the live system kept serving users
3. **Phase 3 (after hours):** quick incremental sync, swap the live path to a junction pointing at the new volume, recreate the share with identical permissions, verify

I picked RAID 5 because it gave the most usable capacity from four drives while still tolerating a single-drive failure, which matched the practice's tolerance for risk and budget.

I used a directory junction (`mklink /J`) for the path swap because the Gendex imaging software was tied to a specific drive path (`E:\Gendex Images`) and had no usable option to point it at a different drive or folder. The application would only read and write images at that exact path, and reconfiguring it to use `D:\` directly was not viable. A junction sidesteps the limitation entirely: at the file system level, `E:\Gendex Images` becomes a transparent pointer to `D:\Gendex Images`, so the application keeps operating on its expected path while the underlying storage physically lives on the new RAID 5 array. From the software's point of view, nothing changed.

## Steps

### 1. Document existing configuration
Before touching anything I captured baseline state so I had a rollback reference:
- Disk Management snapshot of all volumes, sizes, and drive letters
- The exact share configuration on `E:\Gendex Images`: share name, advanced share settings, NTFS and share permissions (Administrators: Full Control; Anon Dental account: Change & Read)
- The Group Policy drive map that pushed `Y:` to staff workstations pointing at `\\<server>\Gendex Images`
- Confirmed the Gendex imaging software was working normally end-to-end

![Existing volume layout in Disk Management before changes](/assets/projects/raid-replacement/02-disk-mgmt-before.jpg)

![Group Policy drive map for the Gendex Images share](/assets/projects/raid-replacement/03-gpo-drive-maps.png)

### 2. Install new drives and create the RAID 5 virtual disk
1. Shut down the server cleanly
2. Installed the four new 1.2TB SAS drives into available bays using the standard Dell hard drive caddies

![New drive seated in caddy ready to insert](/assets/projects/raid-replacement/04-drive-in-caddy.jpg)

3. Powered on and pressed F2 at POST to enter System Setup
4. From System Setup Main Menu, selected **Device Settings**

![Dell EMC System Setup Main Menu](/assets/projects/raid-replacement/05-system-setup-main.jpg)

5. From Device Settings, selected **RAID Controller in Slot 1: Dell PERC <PERC H730P Adapter> Configuration Utility**

![Device Settings with PERC H730P highlighted](/assets/projects/raid-replacement/06-device-settings.jpg)

6. From the PERC Main Menu, opened **Virtual Disk Management** and noted the existing virtual disk's number and RAID level (so I would not confuse it with the new one)

![PERC H730P Configuration Utility Main Menu](/assets/projects/raid-replacement/07-perc-main-menu.jpg)

![Existing Virtual Disk 0: RAID 5 documented before changes](/assets/projects/raid-replacement/08-existing-vdisk.jpg)

7. Backed out and went to **Configuration Management → Create Virtual Disk**

![Configuration Management with Create Virtual Disk highlighted](/assets/projects/raid-replacement/09-config-mgmt.jpg)

8. Set **RAID Level: RAID 5**

![Selecting RAID 5 from the RAID level list](/assets/projects/raid-replacement/10-raid-level.jpg)

![Select Physical Disks screen with all four drives unchecked before selection](/assets/projects/raid-replacement/10b-physical-disks-before.jpg)

9. Used **Select Physical Disks** and checked all four newly installed SAS drives (and only those four) using spacebar

![All four new 1.091TB SAS drives selected](/assets/projects/raid-replacement/11-physical-disks.jpg)

10. **Apply Changes** → "The operation has been performed successfully" → **OK**

![Apply Changes success confirmation screen](/assets/projects/raid-replacement/11b-apply-changes-success.jpg)

11. Scrolled to the bottom and selected **Create Virtual Disk** → checked the **Confirm** box → **Yes**
12. Confirmation again → **OK**
13. Returned to **Virtual Disk Management** and verified both the original and the new virtual disks were listed correctly

![Virtual Disk Management showing both VD0 (1.635TB) and VD1 (3.273TB) listed as Ready](/assets/projects/raid-replacement/11c-vdisk-confirmed.jpg)

14. **Finish** and booted into Windows

### 3. Initialize the new volume
1. Opened Disk Manager

![Disk Management showing new unallocated Disk 1 with right-click New Simple Volume menu](/assets/projects/raid-replacement/13-disk-mgmt-new-disk.jpg)

2. Initialized the new disk as GPT
3. Created a Simple Volume, formatted as NTFS, assigned drive letter `D:`

![New Simple Volume Wizard Format Partition step with NTFS and Data volume label selected](/assets/projects/raid-replacement/14-format-partition.jpg)

4. Verified `D:` was visible and writable

![Disk Management after new volume is initialized and online as Data (A:) 3351.73 GB](/assets/projects/raid-replacement/15-disk-mgmt-after-init.jpg)

### 4. Pre-stage the data with FreeFileSync (during business hours)
With the new volume online, I pre-copied while the practice continued working normally. This dramatically shortened the after-hours cutover window.
1. Installed FreeFileSync on the server
2. Created an FFS config: **Mirror** `E:\Gendex Images` (source) to `D:\Gendex Images` (target)
3. Ran the initial mirror and let it complete in the background
4. Saved the FFS config so I could reuse it for the incremental run later that evening

### 5. Cutover (after hours)
1. Confirmed staff were logged out and the Gendex imaging software was idle
2. Ran a final FFS **Update** pass from `E:\Gendex Images` to `D:\Gendex Images` to catch any new images saved during the day
3. Removed the existing share on the original folder:
   - Right-click `E:\Gendex Images` → **Properties → Sharing → Advanced Sharing**
   - Unchecked **Share this folder** → **Apply → OK → Close**
4. Renamed the original folder to `E:\Gendex Images original` (kept it as a fallback rather than deleting)
5. Opened Command Prompt as Administrator and created the junction:

This made `E:\Gendex Images` a transparent pointer to `D:\Gendex Images`. Anything the Gendex software or the share wrote to `E:\Gendex Images` now physically landed on the new RAID 5 array.
6. Recreated the SMB share on the new `E:\Gendex Images` (junction) with the exact share name and permissions captured in step 1

### 6. Verification
1. Wrote a test file to `E:\Gendex Images` and confirmed it appeared in `D:\Gendex Images` (and vice versa), confirming the junction was working
2. From a workstation, opened the mapped drive and confirmed all images were present and accessible with the expected permissions

![File Explorer showing D: and E: paths with identical contents after junction](/assets/projects/raid-replacement/12-file-explorer-after.png)

3. Launched the Gendex imaging software and confirmed:
   - Existing patient images opened normally
   - New x-rays captured and saved successfully
   - File timestamps and counts on `D:\` matched what the application reported
4. Compared total file count and total size between `E:\Gendex Images original` and `D:\Gendex Images` to confirm the mirror was complete
5. Monitored Event Viewer overnight for any disk, share, or application errors

## Outcome
- Anon Dental migrated to a new RAID 5 SAS array with no workflow change for staff
- The `E:\Gendex Images` path and SMB share name were preserved end-to-end, so no client workstations needed to be touched
- After-hours cutover took roughly [XX] minutes of actual outage; the heavy lifting happened during business hours via FreeFileSync
- `E:\Gendex Images original` was kept as a rollback option for [X weeks] before being archived and removed to reclaim space

## Lessons learned
- Pre-staging with FreeFileSync during normal hours is what kept the actual outage short. Doing the full copy during the cutover window would have stretched it from minutes to hours.
- Directory junctions (`mklink /J`) are a clean way to do storage migrations transparent to the application layer. The app never knew the underlying drive changed.
- Documenting share permissions in detail before unsharing is essential. NTFS plus share permissions can be easy to misconfigure when recreating, especially with service-specific accounts.
- Renaming rather than deleting the original folder gave a cheap rollback path during the verification window.

[Back to home](/)
