# SteamOS Update Mechanism

SteamOS updates are performed largely via standard open-source mechanisms,
tied together with custom logic to handle the OS' unique layout and to give
users increased flexibility.

At a high level, the updater uses [RAUC](https://rauc.io/index.html) to write a complete root partition image into the *opposite* [OS Image](partitions.md#redundant-system-copies), then reboots into that image. The system then clones the contents of the `/var` partition over to the updated image, which by extension will also copy over `/etc` changes.

But you're here because you're interested in the gory details. Let's begin.

## 1. Triggering The Update

Triggering an update from the terminal is done via the `steamos-update-os` command.
While there are multiple other similarly-named commands and utilities on the device,
directly invoking those is not recommended as they can jump into the middle of the
chain of events that comprise an update.

The `steamos-update-os` command support two run modes:

1. `now` triggers an immediate update, prompting the user to restart when complete.
1. `after-reboot` executes `steamos-set-bootmode update-other` to add `systemd.unit=steamos-update-os.target` into the kernel command line in grub, ensuring that the next reboot enters "SteamOS Update Mode". Update Mode is a bare-bones mode that simply runs `steamos-update-os now` and displays a nice progress bar. Execution then continues as though the user chose to update `now`.

`steamos-update-os now` simply runs `steamos-atomupd-client`, which is a basic wrapper around the Python package
`steamosatomupd`, installed via `steamos-atomupd-client-git`.

* Interestingly, this package also contains the source for the update server itself is at `/usr/lib/python3.10/site-packages/steamosatomupd/server.py`.
  * While this is not used by the Deck, it makes hosting a custom update server quite easy!
  * The server uses some neat logic to de-dupe common chunks of data across multiple update files, but this is outside the scope of this project.

The main logic begins at `/usr/lib/python3.10/site-packages/steamosatomupd/client.py`, in the `main` function.

## 2 Configuring Which Update To Install

The first step is to load a series of configuration files, most of which can be overwritten via arguments to `steamos-update-os`:

```
usage: steamos-atomupd-client [-h] [-c FILE] [-q] [-d] [--query-only] [--estimate-download-size] [--manifest-file FILE] [--mk-manifest-file]
                              [--update-file UPDATE_FILE] [--update-from-url UPDATE_FROM_URL] [--update-version UPDATE_VERSION]
                              [--variant VARIANT]

SteamOS Update Client

options:
  -h, --help            show this help message and exit
  -c FILE, --config FILE
                        configuration file (default: /etc/steamos-atomupd/client.conf)
  -q, --quiet           hide output
  -d, --debug           show debug messages
  --query-only          only query if an update is available
  --estimate-download-size
                        Include in the update file the estimated download size for each image candidate
  --manifest-file FILE  manifest file (default: /etc/steamos-atomupd/manifest.json)
  --mk-manifest-file    don't use existing manifest file, make one instead
  --update-file UPDATE_FILE
                        update from given file, instead of downloading it from server
  --update-from-url UPDATE_FROM_URL
                        update to a specific RAUC bundle image
  --update-version UPDATE_VERSION
                        update to a specific buildid version. It will fail if either the update file doesn't contain this buildid or if it
                        requires a base image that is newer than the current one
  --variant VARIANT     use this 'variant' value instead of the one parsed from the manifest file
```

The ultimate goal of all these options is to determine the URL of an [RAUC](https://rauc.io/index.html) update bundle, if one exists. The logic used and precedence between these flags is not obvious, so this is the actual logic from top to bottom:

1. If `--update-from-url` is provided, download and install from that URL and skip the rest of the logic below.
1. If `--update-file` is provided, install from this file and skip the rest of the logic below.
1. If `--mk-manifest-file` is provided, skip all manifest parsing and generate an 'Image' specification based on the `/etc/os-release` file.
1. If `--manifest-file` is provided, read a custom manifest from the provided path. Ignore the default - it's applied later.
1. Otherwise, load a manifest from the `[Host]/Manifest` key in the config file provided via `-c` (or the default).
    * This needs to be a local file path.
    * This key is normally missing in the default config file.
1. Otherwise, load a default manifest from `/etc/steamos-atomupd/manifest.json`
1. Parse the manifest to create an 'Image' specification. Override the 'variant' field using `--variant` if provided.
1. Encode the 'Image' specification as a dictionary and fetch an update file (if any) from the `[Server]/QueryUrl` specified in the config.
    * Credentials to the update server are loaded from `~/.netrc`.
1. Apply `--update-version` restrictions.

At this point, we have a RAUC update bundle. See [this documentation](https://rauc.readthedocs.io/en/latest/using.html#creating-bundles) for more details on RAUC bundles.

## 3. Download and Install Update Via RAUC

RAUC supports multiple ways of downloading update files specified in the bundle. The default (and the one in use) is called [casync](https://github.com/systemd/casync), a Linux utility to sync large files across two systems.

* There is a (disabled) option to use a different syncing mechanism known as [desync](https://github.com/folbricht/desync). This is configured via the `[casync]/use-desync` option in `/etc/rauc/system.conf`. 
  * The custom logic powering this looks quite hairy, so it's probably a good idea to keep that off for now.
  * The logic suggests a future mechanism to only download a delta between the data on disk versus the update, massively decreasing update sizes.

RAUC will be using `/tmp`, and it seems that Valve has hit issues with this before. Just before running RAUC, they increase the `/tmp` ramdisk limits to allow 100% of RAM and up to 1 Billion inodes.

* In practice it uses less than half of the 16GB RAM available, but it's probably best to have some free RAM before starting an update.

RAUC is finally triggered via `rauc install <location_to_RAUC_bundle_file>`.

### 3.1 RAUC Config

RAUC is configured to download update data to `/tmp` and verify files using a public key located in `/etc/rauc/keyring.pem`.

* Self-signed certificate
* `Issuer: C = US, ST = Washington, L = Bellevue, O = Valve Corp, CN = steamdeck-images`

RAUC works on the idea of "[slots](https://rauc.readthedocs.io/en/latest/basic.html#slots)", where each slot is something that can be updated. In this case, a slot is a raw disk partition. Each of the two root partitions are present as slots in this configuration. (There's also a third "dev" slot which we'll ignore.)

RAUC is also in charge of changing which slot the system boots from. Valve has installed a custom bootloader backend, along with pre and post install scripts to perform additional tasks.

* The `pre` script (at `/usr/lib/rauc/pre-install.sh`) is quite simple:
  * Sanity checks the update to ensure it won't cause a UUID collision with the currently mounted system partition.
  * Disables all systemd timers to avoid disruptions.
* The `post` script (at `/usr/lib/rauc/post-install.sh`) is much more complex, and is documented below.
* Some community members have already explored [using these pre/post scripts to customize a system install](https://github.com/icedream/steamos-permanent-mods).

### 3.2 RAUC Bootloader Backend

Much of the magic happens in RAUC itself, but it needs to interact with SteamOS' custom boot mechanism. This is handled by a custom bootloader backend located at `/usr/lib/rauc/bootloader-custom-backend.sh`. This script implements a standard RAUC API and is in charge of:

* Fetching what A/B slot is the "current" system image.
* Configuring what A/B slot the system will reboot into next.
* Fetch whether the last boot was successful on a particular A/B slot.

The bootloader itself is simply an interface to get and set information. The actual logic of what gets updated is handled by RAUC. Between the [RAUC logic](https://rauc.readthedocs.io/en/latest/reference.html#rauc-s-basic-update-procedure) and the bootloader backend, the following behavior arises:

* The update will install into the *opposite* slot of the currently-running system.
* The system will reboot into a previously-working slot if the update resulted in a boot failure.
* The update process clears the kernel cmdline flags set by the `steamos-update-os after-reboot` command to ensure we exit Update Mode on next reboot.

## 4. Final Steps

Final update steps are implemented via the `/usr/lib/rauc/post-install.sh` script. This script:

* Re-enables systemd timers.
* Regenerates files containing the UUIDs of system partitions. Partition UUIDs are update-specific, and change with each update.
* Copies the contents of the `/var` partition for the currently-running OS side into the one for the newly updated side:
  * It first *reformats* the `/var` partition of the newly updated OS side.
  * The contents of the current `/var` partition is then `rsync`ed over to the new partition.
  * This means that any changes to `/var` from this point on will be lost.
* Regenerates GRUB for both Image A and Image B.
* Writes the current set of network configurations to the newly updated slot. It's unknown why this is necessary as `/var` (and by extension `/etc` is already synced).

And we're done! Execution finally returns to `steamos-update-os`. If it detects we're currently in Update Mode, it automatically reboots the system after 5 seconds. Otherwise, it nicely asks the user to reboot.

## Other Notes

* Once triggered, RAUC runs as a system daemon. If the terminal process is aborted via `Ctrl+C`, you may need to run `systemctl stop rauc` or `killall rauc` to actually abort the update.
* A failed update often leaves the running system in an odd state, as some things that were temporarily mounted do not get unmounted. A reboot is often needed before retrying an update.
* Error messages get logged to `journald`. Use `journalctl --unit=rauc` to see them.
* If you enter "Update Mode" when there's no update available, it's theoretically possible to get stuck as the relevant kernel cmd is only cleared upon a successful update.
* If you're playing around in a VM and can't get updates to work, run the following two commands, reboot, and try again.
  * `steamos-chroot --partset A -- mkdir -p /var/lib/overlays/etc/upper/NetworkManager/system-connections/`
  * `steamos-chroot --partset B -- mkdir -p /var/lib/overlays/etc/upper/NetworkManager/system-connections/`

## Misc Outputs

### Sample Manifest

```json
{
  "product": "steamos",
  "release": "holo",
  "variant": "steamdeck",
  "arch": "amd64",
  "version": "snapshot",
  "buildid": "20220425.2",
  "checkpoint": false,
  "estimated_size": 0
}
```

### Sample `/etc/os-release`

```
NAME="SteamOS"
PRETTY_NAME="SteamOS"
VERSION_CODENAME=holo
ID=steamos
ID_LIKE=arch
ANSI_COLOR="1;35"
HOME_URL="https://www.steampowered.com/"
DOCUMENTATION_URL="https://support.steampowered.com/"
SUPPORT_URL="https://support.steampowered.com/"
BUG_REPORT_URL="https://support.steampowered.com/"
LOGO=steamos
VARIANT_ID=steamdeck
BUILD_ID=20220425.2
VERSION_ID=3.1
```
