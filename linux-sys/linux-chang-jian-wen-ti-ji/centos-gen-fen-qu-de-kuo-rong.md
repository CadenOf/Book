# Centos 根分区的扩容

* **查看本机现有分区情况**

```text
[root@linuxidc ~]# df -h
文件系统容量已用可用已用%% 挂载点
/dev/mapper/VolGroup-lv_root   7.7G  7.1G 155M  98% /
tmpfs                          3.9G  296K 3.9G  1% /dev/shm
/dev/sda1                      485M  64M 396M  14% /boot
/dev/sda3                      83G  350M  79G  1% /media
```

* **查看本机的磁盘情况**

```text
[root@linuxidc ~]# fdisk -l
Disk /dev/sda: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054cylinders
Units = cylinders of 16065 * 512 = 8225280bytes
Sector size (logical/physical): 512 bytes /512 bytes
I/O size (minimum/optimal): 512 bytes / 512bytes
Disk identifier: 0x000dc0ad

 Device Boot      Start        End      Blocks  Id  System
/dev/sda1  *          1          64      512000  83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              64        2089  16264192  8e  Linux LVM
/dev/sda3            2090      13054  88076362+  83  Linux

Disk /dev/mapper/VolGroup-lv_root: 8325 MB,8325693440 bytes
255 heads, 63 sectors/track, 1012 cylinders
Units = cylinders of 16065 * 512 = 8225280bytes
Sector size (logical/physical): 512 bytes /512 bytes
I/O size (minimum/optimal): 512 bytes / 512bytes
Disk identifier: 0x00000000
Disk /dev/mapper/VolGroup-lv_swap: 8325 MB,8325693440 bytes
255 heads, 63 sectors/track, 1012 cylinders
Units = cylinders of 16065 * 512 = 8225280bytes
Sector size (logical/physical): 512 bytes /512 bytes
I/O size (minimum/optimal): 512 bytes / 512bytes
Disk identifier: 0x00000000
```

* **添加第二块硬盘后，对其进行分区**

```text
[root@linuxidc ~]# fdisk /dev/sdb
Device contains neither a valid DOSpartition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with diskidentifier 0xfa4abbdc.
Changes will remain in memory only, untilyou decide to write them.
After that, of course, the previous contentwon't be recoverable.
Warning: invalid flag 0x0000 of partitiontable 4 will be corrected by w(rite)
WARNING: DOS-compatible mode is deprecated.It's strongly recommended to
      switch off the mode (command 'c') and change display units to
      sectors (command 'u').
      
Command (m for help): m
Command action
 a  toggle a bootable flag
 b  edit bsd disklabel
 c  toggle the dos compatibilityflag
 d  delete a partition
 l  list known partition types
 m  print this menu
 n  add a new partition
 o  create a new empty DOSpartition table
 p  print the partition table
 q  quit without saving changes
 s  create a new empty Sundisklabel
 t  change a partition's system id
 u  change display/entry units
 v  verify the partition table
 w  write table to disk and exit
 x  extra functionality (expertsonly)
 
Command (m for help): n

Command action
 e  extended
 p  primary partition (1-4)
p  

Partition number (1-4): 1
First cylinder (1-2088, default 1): 
Using default value 1
Last cylinder, +cylinders or +size{K,M,G}(1-2088, default 2088): 
Using default value 2088

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

说明：上面操作对sdb硬盘进行了分区操作，设为sdb4分区了（当然上面建立的主分区可以为1-4中的任意一个，我这里选择的4）。
```

* **对新建立的sdb4分区进行格式化**

```text
[root@linuxidc ~]# mkfs.ext4 /dev/sdb1
mke2fs 1.41.12 (17-May-2010)
文件系统标签=
操作系统:Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
1048576 inodes, 4192957 blocks
209647 blocks (5.00%) reserved for thesuper user
第一个数据块=0
Maximum filesystem blocks=4294967296
128 block groups
32768 blocks per group, 32768 fragments pergroup
8192 inodes per group
Superblock backups stored on blocks: 
      32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
      4096000
正在写入inode表: 完成
Creating journal (32768 blocks): 完成
Writing superblocks and filesystemaccounting information: done完成

This filesystem will be automaticallychecked every 20 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.

说明：将sdb4分区格式化为ext4格式，因为CentOS安装是系统的格式ext4，所以这里要注意一下。
```

{% hint style="info" %}
如果格式化分区报错

\[root@k8s-tooling-master02 salt\]\# mkfs.ext4 /dev/sda3 mke2fs 1.42.9 \(28-Dec-2013\) Could not stat /dev/sda3 --- No such file or directory

The device apparently does not exist; did you specify it correctly?

可执行命令 partprobe， partprobe 可以修改 kernel 中分区表，使 kernel 重新读取分区表。 因此，使用该命令就可以创建分区并且在不重新启动机器的情况下系统能够识别这些分区。
{% endhint %}

* **格式后的sdb4分区添加为物理卷**

```text
[root@linuxidc ~]#   pvcreate      /dev/sdb1
Physical volume "/dev/sdb4" successfully created
```

* **查看当前系统的物理卷（PV）情况** 可以看到添加的新物理卷sdb4，大小都符合我们添加时的设置

```text
[root@linuxidc ~]# pvdisplay
---Physical volume ---
PVName /dev/sda2
VGName VolGroup
PVSize 15.51 GiB / not usable3.00 MiB
Allocatable yes (butfull)
PESize 4.00 MiB
Total PE 3970
Free PE 0
Allocated PE 3970
PVUUID Up77sG-sNKf-Ja0k-crBf-N0wz-a5hy-T6sVFh
"/dev/sdb4" is a new physical volume of "15.99 GiB"
---NEW Physical volume ---
PVName /dev/sdb4
VGName 
PVSize 15.99 GiB
Allocatable NO
PESize 0 
Total PE 0
Free PE 0
Allocated PE 0
PVUUID Gqnw7N-HooX-S2Nz-1NIZ-YpOe-g0jo-grG1rQ 
```

* **查看当前卷组情况**

```text
[root@linuxidc ~]# vgdisplay
---Volume group ---
VGName VolGroup
System ID 
Format lvm2
Metadata Areas 1
Metadata Sequence No 3
VGAccess read/write
VGStatus resizable
MAXLV 0
CurLV 2
Open LV 2
MaxPV 0
CurPV 1
ActPV 1
VGSize 15.51 GiB
PESize 4.00 MiB
Total PE 3970
Alloc PE / Size 3970 / 15.51GiB
Free PE / Size 0 / 0 
VGUUID PPrifh-glzl-nnxX-YkBm-xJ1s-rgnw-d1WsJd
```

* **将分区sdb4转换为扩展分区**

```text
[root@linuxidc ~]# vgextend VolGroup00 /dev/sdb1
Volume group "VolGroup" successfully extended
```

* **查看当前的逻辑卷**

```text
[root@linuxidc ~]# lvdisplay
---Logical volume ---
LVPath /dev/VolGroup00/lv_root
LVName lv_root
VGName VolGroup
LVUUID dCIsej-0NWX-bVFe-bT6L-c4eY-oy4G-9lNBOC
LVWrite Access read/write
LVCreation host, time , 
LVStatus available
#open 1
LVSize 7.75 GiB
Current LE 1985
Segments 1
Allocation inherit
Read ahead sectors auto
-currently set to 256
Block device 253:0

---Logical volume ---
LVPath /dev/VolGroup00/lv_swap
LVName lv_swap
VGName VolGroup
LVUUID K5bxJo-2SmM-6NCU-mBJP-Qzp1-ZODH-nbLK0k
LVWrite Access read/write
LVCreation host, time , 
LVStatus available
#open 1
LVSize 7.75 GiB
Current LE 1985
Segments 1
Allocation inherit
Read ahead sectors auto
-currently set to 256
Block device 253:1

说明：这里可以看到“/”根分区的路径名称为：/dev/VolGroup/lv_root

```

* **查看扩展后的卷组情况**

```text
[root@linuxidc ~]# vgdisplay
---Volume group ---
VGName VolGroup00
System ID 
Format lvm2
Metadata Areas 2
Metadata Sequence No 4
VGAccess read/write
VGStatus resizable
MAXLV 0
CurLV 2
Open LV 2
MaxPV 0
CurPV 2
ActPV 2
VGSize 31.50 GiB
PESize 4.00 MiB
Total PE 8064
Alloc PE / Size 3970 / 15.51GiB
Free PE / Size 4094 / 15.99 GiB
VGUUID PPrifh-glzl-nnxX-YkBm-xJ1s-rgnw-d1WsJd

说明：扩展后的卷子情况，可以看出大小增加了16GB（与第八步中的对比）
```

* **将新增的逻辑卷全部扩展到 `/`分区中**

```text
[root@linuxidc ~]# lvextend -L +15G /dev/VolGroup00/lv_root 
Rounding size to boundary between physical extents: 15.99 GiB
Extending logical volume lv_root to 23.75 GiB
Logical volume lv_root successfully resized

说明：这里sdb4总共有16G，所以把16BG全部添加给根分区。

```

* **查看 `/`根分区格式，并重新刷新根分区的大小**

```text
[root@linuxidc ~]# e2fsck -f /dev/VolGroup00/lv_root 
e2fsck 1.41.12 (17-May-2010)
/dev/VolGroup/lv_root is mounted.
e2fsck: 无法继续, 中止.
[root@linuxidc ~]# resize2fs /dev/VolGroup00/lv_root 
resize2fs 1.41.12 (17-May-2010)
Filesystem at /dev/VolGroup/lv_root ismounted on /; on-line resizing required
old desc_blocks = 1, new_desc_blocks = 2
Performing an on-line resize of/dev/VolGroup/lv_root to 6224896 (4k) blocks.
The filesystem on /dev/VolGroup/lv_root isnow 6224896 blocks long.

说明：命令resize2fs即刷新根分区“/dev/VolGroup/lv_root”的容量。
刷新分区表：partprobe
```

* **查看刷新后根分区的大小**

```text
[root@linuxidc ~]# df -h
文件系统容量                    已用   可用  已用  % 挂载点
/dev/mapper/VolGroup-lv_root  24G   7.1G  16G  32% /
tmpfs                         3.9G  144K  3.9G 1%  /dev/shm
/dev/sda3                     83G   350M  79G  1%  /media/Lucene
```







