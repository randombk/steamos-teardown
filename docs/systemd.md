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

## Appendix

### steamos-customizations - systemd system

```bash
(deck@deck etc)$ grep -r 'steamos-customizations is free software' /usr/lib/systemd/system/
/usr/lib/systemd/system/etc.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/home-swapfile.swap:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/opt.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/root.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/srv.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-boot.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-cfs-debugfs-tunings.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-create-homedir.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-finish-oobe-migration.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-install-grub.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-install-steamcl.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-offload.target:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-settings-importer.service:#  steamos-customizations is free software; you can redistribute it and/or modify
/usr/lib/systemd/system/steamos-update-os-plymouth.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-update-os.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/steamos-update-os.target:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/swapfile.service:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/usr-lib-debug.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/usr-local.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-cache-pacman.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-lib-docker.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-lib-flatpak.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-lib-systemd-coredump.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-log.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/var-tmp.mount:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/rauc.service.d/override.conf:#  steamos-customizations is free software; you can redistribute it and/or
/usr/lib/systemd/system/sddm.service.d/steamos-customizations.conf:#  steamos-customizations is free software; you can redistribute it and/or
```
