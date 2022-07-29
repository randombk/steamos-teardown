# Custom SystemD Units

SteamOS comes packaged with a number of SystemD services, predominantly stored under `/usr/lib/systemd/system`. These services all come from the `steamos-customizations` package:

* `steamos-offload.target`
  * **Name**: `SteamOS Offload`
  * Still no idea what "offload" means
* `steamos-update-os.target`
  * **Name**: `SteamOS Update Mode`
  * It's unclear if this is the trigger for an update, or part of the update chain itself.
* `screen-rotation-quirks.service`
  * **Name**: `Adds or removes device-specific rotation-quirk Xorg configuration`
  * **Runs**: ??? ("`sysinit.target`")
  * Runs `/usr/bin/gpd-xorg-rotation-config`
* `steamos-boot.service`
  * **Name**: `SteamOS Boot Registration`
  * **Runs**: At boot and during the update process. (Oneshot)
  * Runs `/usr/bin/steamos-bootconf set-mode booted` once the system boots up, presumably to update statistics.
* `steamos-cfs-debugfs-tunings.service`
  * **Name**: `set multiple CFS tunings in sched debugfs`
  * **Runs**: At boot. (Oneshot)
  * Runs `/usr/lib/steamos/steamos-cfs-debugfs-settings`.
* `steamos-create-homedir.service`
  * **Name**: `Create UID 1000 homedir on first boot/after factory reset`
  * **Runs**: At boot. (Oneshot)
  * Runs `/usr/lib/steamos/steamos-create-homedir 1000`
* `steamos-devkit-service.service`
  * **Name**: `SteamOS Devkit Service`
  * **Runs**: At boot. (Oneshot)
  * Runs `/usr/lib/steamos/steamos-devkit-service`
  * Lets devs debug software from their PC
  * See https://gitlab.steamos.cloud/devkit
* `steamos-finish-oobe-migration.service`
  * **Name**: `Finish migration from OOBE build`
  * **Runs**: At boot. (Oneshot)
  * Only runs when the `/home/doorstop` directory is present.
* `steamos-glx-backend.service`
  * **Name**: `Choose a GLX backend on boot`
  * **Runs**: At boot. (Oneshot)
  * Runs `/usr/lib/steamos/steamos-glx-backend`
* `steamos-install-grub.service`
  * **Name**: `SteamOS GRUB2 Installation`
  * **Runs**: ??? ("`sysinit.target`")
  * Runs `/usr/bin/grub-install` and `/usr/bin/grub-install`
* `steamos-install-steamcl.service`
  * **Name**: `SteamOS Chainloader Installation`
  * **Runs**: ??? ("`sysinit.target`")
  * Runs `/usr/bin/steamcl-install`
* `steamos-mkvarboot.service`
  * **Name**: `Create SteamOS /var/boot Directory`
  * **Runs**: ???
  * Copies `/boot` to `/var/boot`, excluding `/boot/grub`.
* `steamos-settings-importer.service`
  * **Name**: `SteamOS Settings Importer`
  * **Runs**: At boot. (Oneshot)
  * Runs `/usr/bin/steamos-settings-importer`
* `steamos-update-os-plymouth.service`
  * **Name**: `SteamOS Update Operating System Plymouth`
  * **Runs**: During Update. (Oneshot)
  * Runs `/usr/lib/steamos/steamos-update-os-progress`
* `steamos-update-os.service`
  * **Name**: `SteamOS Update Operating System`
  * **Runs**: During Update. (Oneshot)
  * Runs `/usr/bin/steamos-update-os now`
  * Immediately triggers a system update
