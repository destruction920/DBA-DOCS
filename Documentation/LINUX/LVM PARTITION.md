#LVM PARTITION

######################################################## MANAGING LINUX PARTITION (LVM)

In Linux, pvs, vgs, and lvs are commands used for managing and
displaying information about the Logical Volume Manager (LVM)
components.

(1) PVS (Physical Volumes): The pvs command displays information about
    physical volumes. It provides a quick summary of the physical
    volumes in the system, showing details such as the physical volume
    name, volume group it belongs to, size, and status.

                Example: 
                [root@prd u02]# pvs
                PV             VG   Fmt  Attr PSize   PFree 
                /dev/nvme0n1p3 rhel lvm2 a--   98.99g     0 
                /dev/nvme0n2   psql lvm2 a--  <15.00g 96.00m

(2) VGS (Volume Groups): These are collections of physical volumes that
    create a pool of storage out of which logical volumes can be
    allocated.

               Example:
                [root@prd u02]# vgs
                VG   #PV #LV #SN Attr   VSize   VFree 
                psql   1   1   0 wz--n- <15.00g 96.00m

(3) LVS (Logical Volumes): These are the virtual block devices created
    from the volume groups' storage pool. Logical volumes can be
    resized, moved, and managed independently of the underlying physical
    storage, providing flexibility in managing disk space.

                Example:
                [root@prd u02]# lvs
                LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
                lv   psql -wi-ao---- 14.90g                                                    
                root rhel -wi-ao---- 15.00g                                                    
                swap rhel -wi-ao----  4.00g                                                    
                u01  rhel -wi-ao---- 79.99g  

############################################################### Extend /u02 partition size (Method 1)

Step 1: Check Existing Physical volume \[root@prd u02\]# pvs PV VG Fmt
Attr PSize PFree /dev/nvme0n1 rhel lvm2 a-- 98.99g 0

Step 2: Add a new Hard Disk of 10GB that will be mounted on /u02

Step 3: Create a new Physical Volume pvcreate -v /dev/nvme0n2

Step 4: Create a volume group vgcreate psql /dev/nvme0n2

Step 5: Create a logical volume lvcreate -L 9.9G -n lv psql

Step 6: Format the logical volume mkfs.ext4 /dev/psql/lv

Step 7: Mount the logical volume to a new mount point /u02 mkdir /u02
mount /dev/psql/lv /u02

Step 8: Make entry in /etc/fstab file

Step 9: Add/Extend 5GB Hard Disk in /u02 mount point

Step 10: Resize the existing Physical Volume pvresize /dev/nvme0n2

Step 11: Extend the existing Logical Volume lvextend -L +5G -r
/dev/psql/lv

####################################################################################################################################################### 

############################################################## Extend /u02 partition size (Method 2)

Step 1: Check Existing Physical volume \[root@prd u02\]# pvs PV VG Fmt
Attr PSize PFree /dev/nvme0n1 rhel lvm2 a-- 98.99g 0

Step 2: Add a new Hard Disk of 10GB that will be mounted on /u02

Step 3: Create a new Physical Volume pvcreate -v /dev/nvme0n2

Step 4: Create a volume group vgcreate psql /dev/nvme0n2

Step 5: Create a logical volume lvcreate -L 9.9G -n lv psql

Step 6: Format the logical volume mkfs.ext4 /dev/psql/lv

Step 7: Mount the logical volume to a new mount point /u02 mkdir /u02
mount /dev/psql/lv /u02

Step 8: Make entry in /etc/fstab file

Step 9: Create a Physical Volume of newly added hard disk pvcreate -v
/dev/nvme0n3

Step 10: format the newly added physical volume. mkfs.ext4 /dev/nvme0n3

Step 11: Extend 15GB Harddisk to psql vgextend -v psql /dev/nvme0n3

Step 12: Extend LV lvextend -L +15G -r /dev/psql/lv

#################################################################################################################################################### 

lsof +D /datafiles kill -9 (id) umount /datafiles lvremove
/dev/orclvg/lvdatafiles

NEED TESTING TO DECREASE SIZE OF LVM

lvresize -L NEW_SIZE /dev/volume-group/logical-volume

#################################################################################################################################################### 

IN VM I extend the exiting primary harddisk from 100 GB to 180 GB.

since the OS is installed on this disk (/dev/nvme0n1), and
/dev/nvme0n1p3 is part of the root volume group (rhel), you cannot
delete and recreate partitions casually --- that would be risky and
could lead to data loss or a non-bootable system.

But the good news is: we can safely resize the partition without
deleting anything using growpart, assuming the free space is available
after nvme0n1p3.

1.  Check if growpart is available:

Install the cloud-utils-growpart package if it's not already installed:

yum install cloud-utils-growpart -y

2.  Grow the partition:

Now run:

growpart /dev/nvme0n1 3

This will resize partition 3 (nvme0n1p3) to use all remaining
unallocated space on the disk --- without deleting or touching data.

3.  Resize the physical volume:

pvresize /dev/nvme0n1p3

But I had to no need to resize the pv I just extend the requried lv

4.  Verify:

pvs vgs

this is output of this exercise after partation extend

\[root@RSBDB \~\]# pvs PV VG Fmt Attr PSize PFree /dev/nvme0n1p3 rhel
lvm2 a-- 178.99g 80.00g \[root@RSBDB \~\]# \[root@RSBDB \~\]#
\[root@RSBDB \~\]# vgs VG #PV #LV #SN Attr VSize VFree rhel 1 3 0 wz--n-
178.99g 80.00g \[root@RSBDB \~\]# lvs LV VG Attr LSize Pool Origin Data%
Meta% Move Log Cpy%Sync Convert root rhel -wi-ao---- 15.00g swap rhel
-wi-ao---- 4.00g u01 rhel -wi-ao---- 79.99g \[root@RSBDB \~\]# lvextend
-L +80G -r /dev/rhel/u01 Size of logical volume rhel/u01 changed from
79.99 GiB (20478 extents) to 159.99 GiB (40958 extents). Logical volume
rhel/u01 successfully resized. resize2fs 1.45.4 (23-Sep-2019) Filesystem
at /dev/mapper/rhel-u01 is mounted on /u01; on-line resizing required
old_desc_blocks = 10, new_desc_blocks = 20 The filesystem on
/dev/mapper/rhel-u01 is now 41940992 (4k) blocks long.
