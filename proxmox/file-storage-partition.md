# Partition Disks for File Storage

When Proxmox is first installed, it creates two storage pools - `local` and `local-lvm`.

For a 1TB SSD, Proxmox partitioned
- ~100GB for `local` - the Proxmox Debian root filesystem
- Rest of the space for `local-lvm`

`local` is a file level storage and `local-lvm` is block level storage.

**Block level storage** can be thin provisioned (only the space that is written to is used) and are better for VMs. Not all kinds of files can be stored in these.

**File level storage** is POSIX compatible and can store any kind of file.

Can reduce the size of block level storage `local-lvm` and make space for a new file level storage partition `local-drive` to store files.

`local-drive` can be mounted to the VMs and containers in `local-lvm` for file storage.

> [!CAUTION]  
> Even though it is possible to delete the `local-lvm` block level storage, expand the `local` file level storage to use the entire disk, and store everything (including VMs and containers) in `local`, this approach is NOT recommended.
> 
> Block level storage like `local-lvm` is thin provisioned and specifically optimized for VM and containers. It only uses space as data is written, making it extremely space efficient.
> 
>  A better approach is to dedicate `local-lvm` just for VMs and containers and create a separate file level storage `local-drive` for other files such as media, backups, and more.

### Initial status of `pve/data` thin-pool `local-lvm`

`local-lvm` with three VMs - 100, 101, and 102.

```bash
root@optiplex:~# lsblk
NAME                         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1                      259:0    0 931.5G  0 disk 
├─nvme0n1p1                  259:1    0  1007K  0 part 
├─nvme0n1p2                  259:2    0     1G  0 part /boot/efi
└─nvme0n1p3                  259:3    0 930.5G  0 part 
  ├─pve-swap                 252:0    0     8G  0 lvm  [SWAP]
  ├─pve-root                 252:1    0    96G  0 lvm  /
  ├─pve-data_tmeta           252:2    0   8.1G  0 lvm  
  │ └─pve-data-tpool         252:4    0 794.3G  0 lvm  
  │   ├─pve-data             252:5    0 794.3G  1 lvm  
  │   ├─pve-vm--100--disk--0 252:6    0    32G  0 lvm  
  │   ├─pve-vm--101--disk--0 252:7    0    32G  0 lvm  
  │   └─pve-vm--102--disk--0 252:8    0    32G  0 lvm  
  └─pve-data_tdata           252:3    0 794.3G  0 lvm  
	└─pve-data-tpool         252:4    0 794.3G  0 lvm  
	  ├─pve-data             252:5    0 794.3G  1 lvm  
	  ├─pve-vm--100--disk--0 252:6    0    32G  0 lvm  
	  ├─pve-vm--101--disk--0 252:7    0    32G  0 lvm  
	  └─pve-vm--102--disk--0 252:8    0    32G  0 lvm  
```

```bash
root@optiplex:~# vgs
  VG  #PV #LV #SN Attr   VSize    VFree 
  pve   1   6   0 wz--n- <930.51g 16.00g
```

```bash
root@optiplex:~# lvs
  LV            VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data          pve twi-aotz-- 794.30g             3.07   0.34                            
  root          pve -wi-ao----  96.00g                                                    
  swap          pve -wi-ao----   8.00g                                                    
  vm-100-disk-0 pve Vwi-a-tz--  32.00g data        22.12                                  
  vm-101-disk-0 pve Vwi-a-tz--  32.00g data        26.03                                  
  vm-102-disk-0 pve Vwi-a-tz--  32.00g data        28.00     
```

### Delete the default `pve/data` thin-pool `local-lvm`

> [!WARNING]
> This will delete all the VMs and containers on the `local-lvm` thin-pool.

```bash
lvremove <volume-group>/<logical-volume-name>
```

```bash
root@optiplex:~# lvremove pve/data
Do you really want to remove active logical volume pve/data? [y/n]: y
  Logical volume "data" successfully removed.
```

```bash
root@optiplex:~# fdisk -l
Disk /dev/nvme0n1: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: Samsung SSD 980 PRO 1TB                 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 202AFF1F-A159-49DC-A2AB-F1BE1E75E5D7

Device           Start        End    Sectors   Size Type
/dev/nvme0n1p1      34       2047       2014  1007K BIOS boot
/dev/nvme0n1p2    2048    2099199    2097152     1G EFI System
/dev/nvme0n1p3 2099200 1953525134 1951425935 930.5G Linux LVM

Disk /dev/mapper/pve-swap: 8 GiB, 8589934592 bytes, 16777216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/pve-root: 96 GiB, 103079215104 bytes, 201326592 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes                                      
```

```bash
root@optiplex:~# lsblk
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1      259:0    0 931.5G  0 disk 
├─nvme0n1p1  259:1    0  1007K  0 part 
├─nvme0n1p2  259:2    0     1G  0 part /boot/efi
└─nvme0n1p3  259:3    0 930.5G  0 part 
  ├─pve-swap 252:0    0     8G  0 lvm  [SWAP]
  └─pve-root 252:1    0    96G  0 lvm  /
```

```bash
root@optiplex:~# pvs
  PV             VG  Fmt  Attr PSize    PFree   
  /dev/nvme0n1p3 pve lvm2 a--  <930.51g <826.51g
```

```bash
root@optiplex:~# vgs
  VG  #PV #LV #SN Attr   VSize    VFree   
  pve   1   2   0 wz--n- <930.51g <826.51g
```

```bash
root@optiplex:~# lvs
  LV   VG  Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root pve -wi-ao---- 96.00g                                                    
  swap pve -wi-ao----  8.00g 
```

### Create a new thin-pool `pve/data` with reduced size

Proxmox uses metadata for thin provisioned volumes in VMs and snapshots.

> [!TIP]
> It is better to over-allocate the space for metadata than run out.
> 
> For moderate to heavy usage, 1-5% of the total size of the thin-pool is typical for Proxmox.

Allocating 256GB for the thin-pool and 4GB for metadata.

```bash
root@optiplex:~# lvcreate --type thin-pool -L 256G --poolmetadatasize 4G -n data pve
  Thin pool volume with chunk size 64.00 KiB can address at most <15.88 TiB of data.
  Logical volume "data" created.
```

```bash
root@optiplex:~# lsblk
NAME               MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1            259:0    0 931.5G  0 disk 
├─nvme0n1p1        259:1    0  1007K  0 part 
├─nvme0n1p2        259:2    0     1G  0 part /boot/efi
└─nvme0n1p3        259:3    0 930.5G  0 part 
  ├─pve-swap       252:0    0     8G  0 lvm  [SWAP]
  ├─pve-root       252:1    0    96G  0 lvm  /
  ├─pve-data_tmeta 252:2    0     4G  0 lvm  
  │ └─pve-data     252:4    0   256G  0 lvm  
  └─pve-data_tdata 252:3    0   256G  0 lvm  
    └─pve-data     252:4    0   256G  0 lvm
```

```bash
root@optiplex:~# pvs
  PV             VG  Fmt  Attr PSize    PFree   
  /dev/nvme0n1p3 pve lvm2 a--  <930.51g <562.51g
```

```bash
root@optiplex:~# vgs
  VG  #PV #LV #SN Attr   VSize    VFree   
  pve   1   3   0 wz--n- <930.51g <562.51g
```

```bash
root@optiplex:~# lvs
  LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data pve twi-a-tz-- 256.00g             0.00   0.42                            
  root pve -wi-ao----  96.00g                                                    
  swap pve -wi-ao----   8.00g
```

### Allocate rest of the space for file level storage `pve/drive`

```bash
root@optiplex:~# lvcreate -l 100%FREE -n drive pve
  Logical volume "drive" created.
```

```bash
root@optiplex:~# lsblk
NAME               MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1            259:0    0 931.5G  0 disk 
├─nvme0n1p1        259:1    0  1007K  0 part 
├─nvme0n1p2        259:2    0     1G  0 part /boot/efi
└─nvme0n1p3        259:3    0 930.5G  0 part 
  ├─pve-swap       252:0    0     8G  0 lvm  [SWAP]
  ├─pve-root       252:1    0    96G  0 lvm  /
  ├─pve-data_tmeta 252:2    0     4G  0 lvm  
  │ └─pve-data     252:4    0   256G  0 lvm  
  ├─pve-data_tdata 252:3    0   256G  0 lvm  
  │ └─pve-data     252:4    0   256G  0 lvm  
  └─pve-drive      252:5    0 562.5G  0 lvm
```

```bash
root@optiplex:~# pvs
  PV             VG  Fmt  Attr PSize    PFree
  /dev/nvme0n1p3 pve lvm2 a--  <930.51g    0
```

```bash
root@optiplex:~# vgs
  VG  #PV #LV #SN Attr   VSize    VFree
  pve   1   4   0 wz--n- <930.51g    0 
```

```bash
root@optiplex:~# lvs
  LV    VG  Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data  pve twi-a-tz--  256.00g             0.00   0.42                            
  drive pve -wi-a----- <562.51g                                                    
  root  pve -wi-ao----   96.00g                                                    
  swap  pve -wi-ao----    8.00g 
```

```bash
root@optiplex:~# mkfs.ext4 /dev/pve/drive 
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done                            
Creating filesystem with 147458048 4k blocks and 36872192 inodes
Filesystem UUID: 098a1b6a-37ab-45c8-ba60-5bcd2efe93c8
Superblock backups stored on blocks: 
		32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
		4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
		102400000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

### Mount `/dev/pve/drive` and update `/etc/fstab` to auto-mount it on boot

```bash
root@optiplex:~# mkdir /mnt/drive
root@optiplex:~# mount /dev/pve/drive /mnt/drive
root@optiplex:~# echo "/dev/pve/drive /mnt/drive ext4 defaults 0 0" >> /etc/fstab 
```

### Add the newly created `drive` partition as Directory storage on Proxmox UI

1. Login to Proxmox Web UI
2. Datacenter -> Storage -> Click "Add"
3. Click "Directory"
4. Select 'ID: "local-drive"'
5. Select 'Directory: "/mnt/drive"'
6. Select 'Content: All'
7. Click "Add"

![](/attachments/proxmox-local-drive-storage.png)

## References
[1] https://pve.proxmox.com/pve-docs/chapter-pvesm.html
[2] https://pve.proxmox.com/wiki/Logical_Volume_Manager_(LVM)
[3] https://forum.proxmox.com/threads/local-lvm.116289/post-503227
