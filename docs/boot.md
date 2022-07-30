# Booting SteamOS

**Pre-Reads**:

* [Partition Layout and the Read-Only OS Implementation](docs/partitions.md)

The SteamOS boot process is a three-staged UEFI affair.

## Stage 1: Steam Chainloader

The first EFI entrypoint is located in `/dev/vda1`, which is responsible for chaining to the configured A/B image and maintaining some basic stats about how often each image was booted.

The contents of this boot partition is very simple:

```
/dev/vda1
├── [2.0K]  efi
│   ├── [2.0K]  boot
│   │   ├── [123K]  bootx64.efi         => Initial EFI boot image
│   │   └── [   0]  steamcl-restricted  => Empty file, unknown purpose
│   └── [2.0K]  steamos
│       ├── [123K]  steamcl.efi         => Same file as 'bootx64.efi' above
│       ├── [   0]  steamcl-restricted  => Empty file, unknown purpose
│       ├── [  76]  steamcl-version     => Simple version string plus the SHA256 of 'bootx64.efi'
│       └── [897K]  steamos-bootconf    => Identical to `/usr/bin/steamos-bootconf`
└── [2.0K]  SteamOS
    └── [2.0K]  conf
        ├── [ 233]  A.conf  => Statistics file for Image A
        └── [ 296]  B.conf  => Statistics file for Image B
```

Exactly how this stage remembers which image to boot is currently unknown. The two interesting files, `A.conf` and `B.conf` are fairly simple and do not contain an obvious flag for which one is active. It is possible the image selection is baked into the `bootx64.efi` binary itself. As `steamos-bootconf` is a binary, we'll need to reverse it to extract its secrets.

An example contents of `B.conf` is as follows:

```
boot-requested-at: 0
boot-other: 0
boot-other-disabled: 0
boot-attempts: 0
boot-count: 8
boot-time: 20220728043451
image-invalid: 0
update: 0
update-disabled: 0
update-window-start: 0
update-window-end: 0
loader: 
partitions: 
title: B
comment: [2022-07-27 21:34:51 -0700] bootconf mode: boot-ok
```

## Stage 2: GRUB

The stage 2 EFI image is specific to each image, and is located at `/dev/vda2` (A) and `/dev/vda3` (B). The files here are generated via `/usr/bin/grub-mkimage`. This stage contains grub and little else:

```
/dev/vda2
├── [2.0K]  EFI
│   └── [2.0K]  steamos
│       ├── [5.5K]  grub.cfg        => GRUB configuration
│       └── [656K]  grubx64.efi     => GRUB EFI image
└── [2.0K]  SteamOS                 
    └── [2.0K]  partsets            => Text files containing disk UUIDs
        ├── [ 126]  A
        ├── [ 347]  all
        ├── [ 126]  B
        ├── [ 126]  other
        ├── [ 126]  self
        └── [  83]  shared
```

The grub config is mostly standard and contains a single boot option (The 'Advanced Options' menu is the exact same as the main entry):

```
menuentry 'SteamOS' --class steamos --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-293b8cff-e33d-4aad-9385-d4be2891df75' {
	recordfail
	load_video
	gfxmode $linux_gfx_mode
	insmod steamenv
	insmod gzio
	if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
	insmod part_gpt
	insmod btrfs
	search --no-floppy --fs-uuid --set=root 293b8cff-e33d-4aad-9385-d4be2891df75
	steamenv_boot	linux /boot/vmlinuz-linux-neptune console=tty1 rd.luks=0 rd.lvm=0 rd.md=0 rd.dm=0 rd.systemd.gpt_auto=no loglevel=3 quiet splash plymouth.ignore-serial-consoles module_blacklist=tpm amd_iommu=off amdgpu.gttsize=8128 spi_amd.speed_dev=1 audit=0 fbcon=vc:4-6 fbcon=rotate:1
	initrd	/boot/amd-ucode.img /boot/initramfs-linux-neptune.img
}
```

One interesting note is the `steamenv_boot` command, which is a non-standard extension presumably implemented directly inside `grubx64.efi`.

The files in the `SteamOS/partsets` directory contain the disk partition UUIDs for each partition. How these are used during boot is unknown. For instance, the `all` file contains:

```
esp 910fa777-03cd-7142-9f3f-bead6d122f49
efi-A 2702b34c-84c1-ec4c-af28-a4cff0c23bf9
efi-B 2a97b5d2-7392-774a-a168-1429b98e7982
rootfs-A eac2d34a-cd2e-354e-ac0a-6e6121ed8535
rootfs-B 428fb1e5-c1e4-de4c-ac22-d46b3167c7bc
var-A 5da106b6-3fe1-1941-b935-6ade27da6f1b
var-B 0357eecb-944b-c446-b76b-3781832d918c
home 997a28d3-92e9-8b41-ab97-72b9bdb55386
```

## Stage 3: Linux Kernel

At last, we get to boot the Linux kernel itself. The `initramfs` and kernel image is stored on `/boot` on the root partition. There's a symlink `/boot/efi => /efi` linking to the Stage 2 GRUB partition of the selected A/B image.

* Initramfs path: `/boot/initramfs-linux-neptune.img`
* Kernel path: `/boot/vmlinuz-linux-neptune`

Initramfs contains the following hooks (configured at `/etc/ostree-mkinitcpio.conf`):

```
HOOKS="base systemd ostree autodetect modconf block filesystems keyboard fsck"
```