```
[root@host-192-168-3-6 ~]# lvm
lvm> pvc
pvchange  pvck      pvcreate
lvm> pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
lvm> lvm
lvmchange    lvmconfig    lvmdiskscan  lvmsadc      lvmsar
lvm> lvc
lvchange   lvconvert  lvcreate
lvm> vgcreate vgdata
  No command with matching syntax recognised.  Run 'vgcreate --help' for more information.
  Correct command syntax is:
  vgcreate VG_new PV ...

lvm> vgcreate vgdata /dev/vdb
  Volume group "vgdata" successfully created
lvm> vgdisplay
  --- Volume group ---
  VG Name               vgdata
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <350.00 GiB
  PE Size               4.00 MiB
  Total PE              89599
  Alloc PE / Size       0 / 0
  Free  PE / Size       89599 / <350.00 GiB
  VG UUID               Et2eCf-amVd-wDln-6NK1-IDza-YcnC-8XC2Dk

  --- Volume group ---
  VG Name               centos
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <49.00 GiB
  PE Size               4.00 MiB
  Total PE              12543
  Alloc PE / Size       12543 / <49.00 GiB
  Free  PE / Size       0 / 0
  VG UUID               3sPMh3-cxGR-wlD3-AhL4-uGfZ-HmCj-kmrPwU

lvm> lvcreate -L 100G -n dockerdir vgdaata
  Volume group "vgdaata" not found
  Cannot process volume group vgdaata
lvm> lvcreate -L 100G -n dockerdir vgdata
  Logical volume "dockerdir" created.
lvm> lvcreate -L 100G -n harbordir  vgdata
  Logical volume "harbordir" created.
lvm> lvcreate -L 120G -n nfsdir  vgdata
  Logical volume "nfsdir" created.
lvm> lvdisplay vgdata
  --- Logical volume ---
  LV Path                /dev/vgdata/dockerdir
  LV Name                dockerdir
  VG Name                vgdata
  LV UUID                NxYNDV-TS7Q-igO3-um3Q-ByHk-GZv2-tomFS0
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:17:48 +0800
  LV Status              available
  # open                 0
  LV Size                100.00 GiB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

  --- Logical volume ---
  LV Path                /dev/vgdata/harbordir
  LV Name                harbordir
  VG Name                vgdata
  LV UUID                m1lNfZ-QYqv-Lo1n-hTSu-WBy9-yAuv-fGsfNA
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:18:29 +0800
  LV Status              available
  # open                 0
  LV Size                100.00 GiB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:3

  --- Logical volume ---
  LV Path                /dev/vgdata/nfsdir
  LV Name                nfsdir
  VG Name                vgdata
  LV UUID                D02GtI-MPdw-Tzuz-tKmD-1TTc-pPOo-j4vGRk
  LV Write Access        read/write
  LV Creation host, time host-192-168-3-6, 2019-05-27 11:18:47 +0800
  LV Status              available
  # open                 0
  LV Size                120.00 GiB
  Current LE             30720
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:4

lvm> exit
  Exiting.
[root@host-192-168-3-6 ~]# mkfs.ext4 /dev/vgdata/dockerdir
mke2fs 1.42.9 (28-Dec-2013)
文件系统标签=
OS type: Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
6553600 inodes, 26214400 blocks
1310720 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=2174746624
800 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: 完成
正在写入inode表: 完成
Creating journal (32768 blocks): 完成
Writing superblocks and filesystem accounting information: 完成

[root@host-192-168-3-6 ~]# mkfs.ext4 /dev/vgdata/nfsdir
mke2fs 1.42.9 (28-Dec-2013)
文件系统标签=
OS type: Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
7864320 inodes, 31457280 blocks
1572864 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=2178940928
960 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: 完成
正在写入inode表: 完成
Creating journal (32768 blocks): 完成
Writing superblocks and filesystem accounting information: 完成

[root@host-192-168-3-6 ~]# mkfs.ext4 /dev/vgdata/harbordir
mke2fs 1.42.9 (28-Dec-2013)
文件系统标签=
OS type: Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
6553600 inodes, 26214400 blocks
1310720 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=2174746624
800 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: 完成
正在写入inode表: 完成
Creating journal (32768 blocks): 完成
Writing superblocks and filesystem accounting information: 完成

[root@host-192-168-3-6 ~]# mkdir /nfsdata
[root@host-192-168-3-6 ~]# mkdir /harbordata
[root@host-192-168-3-6 ~]# mkdir /dockerdata
[root@host-192-168-3-6 ~]# vim /etc/fstab
[root@host-192-168-3-6 ~]# mount
mount       mountpoint
[root@host-192-168-3-6 ~]# mount -a
[root@host-192-168-3-6 ~]# df -h
文件系统                      容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root        44G  1.3G   43G    3% /
devtmpfs                       16G     0   16G    0% /dev
tmpfs                          16G     0   16G    0% /dev/shm
tmpfs                          16G  8.5M   16G    1% /run
tmpfs                          16G     0   16G    0% /sys/fs/cgroup
/dev/vda1                    1014M  142M  873M   14% /boot
/dev/mapper/vgdata-dockerdir   99G   61M   94G    1% /dockerdata
/dev/mapper/vgdata-nfsdir     118G   61M  112G    1% /nfsdata
/dev/mapper/vgdata-harbordir   99G   61M   94G    1% /harbordata
[root@host-192-168-3-6 ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Apr 26 16:10:55 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=4ec1bdd6-fcd8-48c5-ab4e-f905e9e4df27 /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/vgdata/dockerdir /dockerdata ext4 defaults 0 0
/dev/vgdata/nfsdir /nfsdata ext4 defaults 0 0
/dev/vgdata/harbordir /harbordata ext4 defaults 0 0

```
