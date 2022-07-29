# Custom System Scripts

SteamOS comes with a large number of handy utility scripts located under `/usr/bin`. These scripts all come from the `steamos-customizations` package:

## Logging & Debug Utilities

* `/usr/bin/steamos-debug`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-critical`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-emergency`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-error`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-info`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-fatal`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-notice`
  * Writes a message and optionally shows a message to the splash screen.
* `/usr/bin/steamos-warning`
  * Writes a message and optionally shows a message to the splash screen.

## Misc System Management

* `/usr/bin/gpd-xorg-rotation-config`
  * Writes `/etc/X11/xorg.conf.d/40-rotation-quirk.conf` if device type equals certain specs, otherwise it deletes the file.
  * This might be a leftover artifact from a dev who was testing on a ThinkPad X220 😊
* `/usr/bin/grub-install`
  * Shortcut to a standard `grub-install` command with custom args
* `/usr/bin/grub-mkimage`
  * Generates the GRUB boot files (See [here](boot.md))
* `/usr/bin/mkswapfile`
  * Creates a swapfile at `$1` with size `$2` (in MB)
* `/usr/bin/mount.steamos`
  * Utility script to mount SteamOS partitions to a target location for a given disk and partition set (A/B)
* `/usr/bin/steamos-boot-install`
* `/usr/bin/steamos-chroot`
* `/usr/bin/steamos-factory-reset`
* `/usr/bin/steamos-factory-reset-config`
  * Creates a config file for factory reset, containing UUIDs of the relevant partitions
* `/usr/bin/steamos-finalize-install`
* `/usr/bin/steamos-halt`
* `/usr/bin/steamos-logger`
* `/usr/bin/steamos-ls`
* `/usr/bin/steamos-mount`
* `/usr/bin/steamos-partsets`
* `/usr/bin/steamos-poweroff`
* `/usr/bin/steamos-readonly`
* `/usr/bin/steamos-reboot`
* `/usr/bin/steamos-set-bootmode`
* `/usr/bin/steamos-settings-importer`
* `/usr/bin/steamos-update-os`
* `/usr/bin/steamos-verity`
* `/usr/bin/update-dracut`
* `/usr/bin/update-grub`
