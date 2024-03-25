---
title: ubuntu 22.04 虚拟机磁盘扩容
date: 2024-03-22 20:43:12
tag: [虚拟化]
category: [工具使用]
excerpt: ubuntu 22.04磁盘扩容操作
---

## 一，Host侧
- 增加虚拟磁盘容量
```
$ sudo qemu-img resize ubuntu22.qcow2 +32G
```
- 备份磁盘
 
```
$ sudo qemu-img create -b ubuntu22.qcow2 -F qcow2 -f qcow2 ubuntu22-test2.qcow2
```
- 修改虚机配置，为防止数据丢失，先从备份的磁盘中演练
```
$ virsh edit ubuntu22
```


## 二，Guest侧
### 2.1 查看当前状态，根目录已经快满了
``` sh
~$ df -TH
Filesystem                Type   Size  Used Avail Use% Mounted on
tmpfs                     tmpfs  1.1G  1.6M  1.1G   1% /run
/dev/mapper/vgubuntu-root ext4    63G   60G  489M 100% /
tmpfs                     tmpfs  5.3G     0  5.3G   0% /dev/shm
tmpfs                     tmpfs  5.3M     0  5.3M   0% /run/lock
tmpfs                     tmpfs  4.2M     0  4.2M   0% /sys/fs/cgroup
/dev/vda1                 vfat   536M  6.4M  530M   2% /boot/efi
tmpfs                     tmpfs  1.1G   70k  1.1G   1% /run/user/1000
tmpfs                     tmpfs  1.1G   66k  1.1G   1% /run/user/128
```
### 2.2 查看当前硬盘信息
/dev/vda已经显示扩容成功，有96 GiB  
但/dev/vda2和/dev/mapper/vgubuntu-root只有不到60G
```
$ sudo fdisk -l

GPT PMBR size mismatch (134217727 != 201326591) will be corrected by write.
The backup GPT table is not on the end of the device.


Disk /dev/vda: 96 GiB, 103079215104 bytes, 201326592 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: E80679B8-CF0C-4496-A391-C734EB923502

Device       Start       End   Sectors  Size Type
/dev/vda1     2048   1050623   1048576  512M EFI System
/dev/vda2  1050624 134215679 133165056 63.5G Linux LVM


Disk /dev/mapper/vgubuntu-root: 59.84 GiB, 64256737280 bytes, 125501440 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```

上面还报`GPT PMBR size mismatch (134217727 != 201326591) will be corrected by write.`错误

### 2.3 修复GPT错误

```
$ sudo parted -l
Kernel not configured for semaphores (System V IPC). Not using udev synchronisation code.
Model: Linux device-mapper (linear) (dm)
Disk /dev/mapper/vgubuntu-swap_1: 3922MB
Sector size (logical/physical): 512B/512B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system     Flags
 1      0.00B  3922MB  3922MB  linux-swap(v1)


Model: Linux device-mapper (linear) (dm)
Disk /dev/mapper/vgubuntu-root: 64.3GB
Sector size (logical/physical): 512B/512B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  64.3GB  64.3GB  ext4


Warning: Not all of the space available to /dev/vda appears to be used, you can
fix the GPT to use all of the space (an extra 67108864 blocks) or continue with
the current setting?
Fix/Ignore?
Fix/Ignore? fix
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 103GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  538MB   537MB   fat32              boot, esp
 2      538MB   68.7GB  68.2GB                     lvm

```
重新用fdisk -l查看错误消失


### 2.4 /dev/vda2扩容

```
$ sudo growpart /dev/vda 2
CHANGED: partition=2 start=1050624 old: size=133165056 end=134215680 new: size=200275935 end=201326559
$ sudo resize2fs /dev/vda2
resize2fs 1.46.5 (30-Dec-2021)
resize2fs: Device or resource busy while trying to open /dev/vda2
Couldn't find valid filesystem superblock.
```

扩容后发现/dev/vda2已增长，但/dev/mapper/vgubuntu-root不变
```
$ sudo fdisk -l
Disk /dev/vda: 96 GiB, 103079215104 bytes, 201326592 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: E80679B8-CF0C-4496-A391-C734EB923502

Device       Start       End   Sectors  Size Type
/dev/vda1     2048   1050623   1048576  512M EFI System
/dev/vda2  1050624 201326558 200275935 95.5G Linux LVM


Disk /dev/mapper/vgubuntu-root: 59.84 GiB, 64256737280 bytes, 125501440 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```

### 2.5 /dev/mapper/vgubuntu-root 扩容

```
$ sudo lvextend -l +100%FREE /dev/mapper/vgubuntu-root
  Size of logical volume vgubuntu/root changed from 59.84 GiB (15320 extents) to 91.84 GiB (23512 extents).
  Kernel not configured for semaphores (System V IPC). Not using udev synchronisation code.
  Logical volume vgubuntu/root successfully resized.
$ sudo resize2fs /dev/mapper/vgubuntu-root
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/vgubuntu-root is mounted on /; on-line resizing required
old_desc_blocks = 8, new_desc_blocks = 12
The filesystem on /dev/mapper/vgubuntu-root is now 24076288 (4k) blocks long.

```

## 三，扩容成功

```
$ df -h
Filesystem                 Size  Used Avail Use% Mounted on
tmpfs                      994M  1.5M  992M   1% /run
/dev/mapper/vgubuntu-root   91G   56G   31G  65% /
tmpfs                      4.9G     0  4.9G   0% /dev/shm
tmpfs                      5.0M     0  5.0M   0% /run/lock
tmpfs                      4.0M     0  4.0M   0% /sys/fs/cgroup
/dev/vda1                  511M  6.1M  505M   2% /boot/efi
tmpfs                      994M   68K  993M   1% /run/user/1000
tmpfs                      994M   64K  993M   1% /run/user/128
```
