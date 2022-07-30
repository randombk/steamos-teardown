# Partition Layout and the Read-Only OS Implementation

## Two System Copies

**SteamOS is comprised of two full copies, called 'Image A' and 'Image B'.**

The system will have one of the images chosen as the 'active' side, and will consistently boot from that image until it is told to swap via `steamos-bootconf`. Each image has its own set of system root, EFI, and configuration partitions. The `/home` partition is shared across both sides, though files in the `/var` partition are synced under normal operation.

* Which image is currently active can be found using `steamos-bootconf list-images` or `findmnt -no partlabel /`.
* We can reboot into the 'other' image using the `steamos-set-bootmode reboot-other` command. However, this is not recommended.
  * Unlike most other SteamOS commands, this one is a binary application as it's a `suid` application.
* We see how this setup maps to the idea of "slots" in the [RAUC Update mechanism used by SteamOS](system-updates.md).

This setup is ultimately there to support the [SteamOS Atomic Update mechanism](system-updates.md), and despite initial appearances is *not* there for redundancy.

## Partition Table

SteamOS is divided into eight partitions, laid out in the following fashion:

```
/dev/vda1 => EFI (early boot) [VFAT, 64MB, Not Mounted by Default]

OS Image A:
    /dev/vda2 => EFI (A) [VFAT, 32MB, Not Mounted by Default]
    /dev/vda4 => Root (A) [BTRFS, 5GB, RO]
    /dev/vda6 => /var (A) [EXT4, 256MB, RW]

OS Image B:
    /dev/vda3 => EFI (B) [VFAT, 32MB, Not Mounted by Default]
    /dev/vda5 => Root (B) [BTRFS, 5GB, RO]
    /dev/vda7 => /var (B) [EXT4, 256MB, RW]

/dev/vda8 => /home [EXT4, <remaining space>, RW]
    Numerous system directories are bind mounted to folders under /home/.steamos:
    -> /opt                         => /home/.steamos/offload/opt
    -> /root                        => /home/.steamos/offload/root
    -> /srv                         => /home/.steamos/offload/srv
    -> /var/cache/pacman            => /home/.steamos/offload/var/cache/pacman
    -> /var/lib/docker              => /home/.steamos/offload/var/lib/docker
    -> /var/lib/flatpak             => /home/.steamos/offload/var/lib/flatpak
    -> /var/lib/systemd/coredump    => /home/.steamos/offload/var/lib/systemd/coredump
    -> /var/log                     => /home/.steamos/offload/var/log
    -> /var/tmp                     => /home/.steamos/offload/var/tmp
```

## Read-Only OS

**`/home` is the preferred safe and persistent storage (or one of the 'offload'-bound paths pointing to `/home`).**

* This means that user-specific configuration, application settings, documents, etc are safely stored and will not be removed upon system updates.
* Flatpaks are stored in `/var/lib/flatpak` and will be similarly persisted.
* Software installed into `/opt` will also be persisted, presenting a potential avenue for installing non-flatpak applications.
* Interestingly, `/var/lib/docker` is also persisted despite Docker and its alternatives not being installed.
* Logs and coredumps are persisted if they are saved to the appropriate locations in `/var/lib`. `/tmp` is a ramdisk and will be cleared on reboot.

**Despite initial appearances, `/var` is persistent**

* During system update, there is a post-update hook to sync the two `/var` partitions.
* Changes made to `/var` in the time between a successful system update and a reboot are at risk of being deleted (except for the bind-mounted directories).

**`/etc` is writable.**

* `/etc` is implemented as an `overlayfs` overlay between the read-only root partition and the `/var/lib/overlays/etc/upper` directory, which is a writable partition.
* See the section on `/var` for warnings about when changes might get lost.

**Writing to the root partition is possible via the `steamos-readonly` utility**, which is a wrapper around standard Linux commands to remount a BTRFS root as RW or RO. However as Valve has consistently pointed out, any changes here is *will* be lost upon system updates.

## OSTree

It is not entirely clear whether SteamOS uses `ostree`. There are several indications that it does (i.e. `/etc/ostree-mkinitcpio.conf`), but it does not use standard update mechanisms (`/etc/ostree` is entirely empty).

## Misc Outputs

### `fdisk`:

```
Disk /dev/vda: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 2EBDF9DF-1584-0042-8EBF-DDC54D426457

Device        Start       End   Sectors  Size Type
/dev/vda1      2048    133119    131072   64M EFI System
/dev/vda2    133120    198655     65536   32M Microsoft basic data
/dev/vda3    198656    264191     65536   32M Microsoft basic data
/dev/vda4    264192  10749951  10485760    5G Linux root (x86-64)
/dev/vda5  10749952  21235711  10485760    5G Linux root (x86-64)
/dev/vda6  21235712  21759999    524288  256M Linux variable data
/dev/vda7  21760000  22284287    524288  256M Linux variable data
/dev/vda8  22284288 134217694 111933407 53.4G Linux home
```

### `findmnt`:

```
TARGET                       SOURCE     FSTYPE     OPTIONS
/                            /dev/vda5  btrfs      rw,relatime,space_cache=v2,subvolid=5,subvol=/
├─/proc                      proc       proc       rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc systemd-1  autofs     rw,relatime,fd=30,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=16444
├─/sys                       sysfs      sysfs      rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security     securityfs securityfs rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup           cgroup2    cgroup2    rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot
│ ├─/sys/fs/pstore           pstore     pstore     rw,nosuid,nodev,noexec,relatime
│ ├─/sys/firmware/efi/efivars
│ │                          efivarfs   efivarfs   rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/bpf              none       bpf        rw,nosuid,nodev,noexec,relatime,mode=700
│ ├─/sys/kernel/debug        debugfs    debugfs    rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/tracing      tracefs    tracefs    rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/config       configfs   configfs   rw,nosuid,nodev,noexec,relatime
│ └─/sys/fs/fuse/connections fusectl    fusectl    rw,nosuid,nodev,noexec,relatime
├─/dev                       devtmpfs   devtmpfs   rw,nosuid,size=4032420k,nr_inodes=1008105,mode=755,inode64
│ ├─/dev/shm                 tmpfs      tmpfs      rw,nosuid,nodev,inode64
│ ├─/dev/pts                 devpts     devpts     rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000
│ ├─/dev/mqueue              mqueue     mqueue     rw,nosuid,nodev,noexec,relatime
│ └─/dev/hugepages           hugetlbfs  hugetlbfs  rw,relatime,pagesize=2M
├─/run                       tmpfs      tmpfs      rw,nosuid,nodev,size=1627980k,nr_inodes=819200,mode=755,inode64
│ └─/run/user/1000           tmpfs      tmpfs      rw,nosuid,nodev,relatime,size=813988k,nr_inodes=203497,mode=700,uid=1000,gid=1000,inode64
├─/efi                       systemd-1  autofs     rw,relatime,fd=46,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=14555
├─/esp                       systemd-1  autofs     rw,relatime,fd=47,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=14557
├─/var                       /dev/vda7  ext4       rw,relatime
│ ├─/var/cache/pacman        /dev/vda8[/.steamos/offload/var/cache/pacman]
│ │                                     ext4       rw,relatime
│ ├─/var/lib/docker          /dev/vda8[/.steamos/offload/var/lib/docker]
│ │                                     ext4       rw,relatime
│ ├─/var/lib/flatpak         /dev/vda8[/.steamos/offload/var/lib/flatpak]
│ │                                     ext4       rw,relatime
│ ├─/var/lib/systemd/coredump
│ │                          /dev/vda8[/.steamos/offload/var/lib/systemd/coredump]
│ │                                     ext4       rw,relatime
│ ├─/var/log                 /dev/vda8[/.steamos/offload/var/log]
│ │                                     ext4       rw,relatime
│ └─/var/tmp                 /dev/vda8[/.steamos/offload/var/tmp]
│                                       ext4       rw,relatime
├─/home                      /dev/vda8  ext4       rw,relatime
├─/etc                       overlay    overlay    rw,relatime,lowerdir=/sysroot/etc,upperdir=/sysroot/var/lib/overlays/etc/upper,workdir=/sysroot/var/lib/overlays/etc/wor
├─/opt                       /dev/vda8[/.steamos/offload/opt]
│                                       ext4       rw,relatime
├─/root                      /dev/vda8[/.steamos/offload/root]
│                                       ext4       rw,relatime
├─/srv                       /dev/vda8[/.steamos/offload/srv]
│                                       ext4       rw,relatime
└─/tmp                       tmpfs      tmpfs      rw,nosuid,nodev,size=4069944k,nr_inodes=1048576,inode64
  └─/tmp/a                   /dev/vda1  vfat       rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro
```
