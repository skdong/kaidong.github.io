# centos vm lvm 扩容

## 添加硬盘后centos系统需要识别scsi硬盘
识别方式：
查看总线号

```bash
[root@controller ~]# ls /sys/class/scsi_host/
host0  host1  host2
#重新扫描scsi总线设备
    
[root@controller ~]# echo "- - -" > /sys/class/scsi_host/host0/scan
[root@controller ~]# echo "- - -" > /sys/class/scsi_host/host1/scan
[root@controller ~]# echo "- - -" > /sys/class/scsi_host/host2/scan
```

## 分区

通过fdisk -l查看当前磁盘,找到添加的磁盘
    
```bash
[root@controller ~]# fdisk -l
Disk /dev/sda: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000c4b34
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    62914559    30944256   8e  Linux LVM
Disk /dev/mapper/centos-root: 29.5 GB, 29490151424 bytes, 57597952 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

## 添加逻辑卷   

为新添加的/dev/sdb硬盘分区，这里全部分为扩展分区，之后将扩展分区全部分为逻辑分区，最后将逻辑分区属性通过t选项改为linux lvm

```bash
[root@controller ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x4aed000b.
Command (m for help): p
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4aed000b
   Device Boot      Start         End      Blocks   Id  System
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): e
Partition number (1-4, default 1): 2
First sector (2048-209715199, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199): 
Using default value 209715199
Partition 2 of type Extended and of size 100 GiB is set
Command (m for help): p
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4aed000b
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb2            2048   209715199   104856576    5  Extended
Command (m for help): n
Partition type:
   p   primary (0 primary, 1 extended, 3 free)
   l   logical (numbered from 5)
Select (default p): l
Adding logical partition 5
First sector (4096-209715199, default 4096): 
Using default value 4096
Last sector, +sectors or +size{K,M,G} (4096-209715199, default 209715199): 
Using default value 209715199
Partition 5 of type Linux and of size 100 GiB is set
Command (m for help): p
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4aed000b
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb2            2048   209715199   104856576    5  Extended
/dev/sdb5            4096   209715199   104855552   83  Linux
Command (m for help): t
Partition number (2,5, default 5): 5
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'
Command (m for help): p
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x4aed000b
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb2            2048   209715199   104856576    5  Extended
/dev/sdb5            4096   209715199   104855552   8e  Linux LVM
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
Syncing disks.
```

## 添加逻辑卷管理  

创建pv，将创建的pv添加到已有的pg中，修改需要扩展的lv大小，当文件系统为xfs时使用xfs_growfs，否则使用resize2fs  -f lvm命令
    
```bash
[root@controller ~]# pvcreate /dev/sdb5
  Physical volume "/dev/sdb5" successfully created
[root@controller ~]# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "centos" using metadata type lvm2
[root@controller ~]# vgextend centos /dev/sdb5
  Volume group "centos" successfully extended
[root@controller ~]# vgdisplay 
  --- Volume group ---
  VG Name               centos
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               129.50 GiB
  PE Size               4.00 MiB
  Total PE              33153
  Alloc PE / Size       7543 / 29.46 GiB
  Free  PE / Size       25610 / 100.04 GiB
  VG UUID               exer5V-Abx9-4xi9-g2Dx-d9My-naUa-Wjqpoc
[root@controller ~]# lvdisplay 
  --- Logical volume ---
  LV Path                /dev/centos/root
  LV Name                root
  VG Name                centos
  LV UUID                DDAigT-wIaH-0Mql-e32w-ilRr-ygN0-7fFbYm
  LV Write Access        read/write
  LV Creation host, time localhost, 2016-04-10 22:38:23 +0800
  LV Status              available
  # open                 1
  LV Size                27.46 GiB
  Current LE             7031
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
   
  --- Logical volume ---
  LV Path                /dev/centos/swap
  LV Name                swap
  VG Name                centos
  LV UUID                l3Z3cv-Qkne-Q8q1-yGvO-wyww-fuNS-F1coc7
  LV Write Access        read/write
  LV Creation host, time localhost, 2016-04-10 22:38:23 +0800
  LV Status              available
  # open                 2
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1
[root@controller ~]# lvextend -L 100G /dev/centos/root
  Size of logical volume centos/root changed from 27.46 GiB (7031 extents) to 100.00 GiB (25600 extents).
  Logical volume root successfully resized.
[root@controller ~]# xfs_growfs /dev/centos/root
meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=1799936 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=7199744, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=3515, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 7199744 to 26214400
[root@controller ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  100G   23G   78G  23% /
devtmpfs                 1.9G     0  1.9G   0% /dev
tmpfs                    1.9G     0  1.9G   0% /dev/shm
tmpfs                    1.9G   17M  1.9G   1% /run
tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda1                497M  164M  334M  33% /boot
tmpfs                    378M     0  378M   0% /run/user/0
```

## 注：
当创建pv时如果提示一下警告

WARNING: Device for PV aaaaa-bbbbb-ccccc-ddd-3243-ddd-eeeee" not found or rejected by a filter.

说明lvm对块设备盘符有过滤器，修改配置文件/etc/lvm/lvm.conf中的filter项 例如：filter = [ "a|.*/|" ]