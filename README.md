# steamos-teardown

An **UNOFFICIAL** repository of information about SteamOS 3, the system powering the Valve Steam Deck. **Pull Requests Are Welcome!**

* The contents of this repository are not endorsed or verified by Valve in any way.
* I make no guarantees of accuracy or correctness:
  * Follow the advice and instructions in this repository at your own risk.
  * SteamOS is a moving target, and the contents of this repository may not necessarily match the current release.
* This repository is aimed at a technical audience already familiar with Linux system administration, particularly on Arch.
* All materials are based on public information and by inspection of a running SteamOS install.
* All material and opinions expressed are solely my own and do not express the views or opinions of my employer.

**The focus is on SteamOS. The Steam Client is proprietary software and is out of scope for this repository.**

## Topics

### SteamOS Customizations

SteamOS 3 is built upon [Arch Linux](https://wiki.archlinux.org/), but applies heavy customizations that deviate from a plain Arch install in significant ways. This section aims to document these differences for any tinkers interested in customizing the OS.

* [Partition Layout and the Read-Only OS Implementation](docs/partitions.md)
* [Booting SteamOS](docs/boot.md)
* [SteamOS Update Mechanism](docs/system-updates.md)
* System Folders, Scripts, and Services
  * [Custom System Scripts](docs/scripts.md)
  * [Custom SystemD Units](docs/systemd.md)
* [Default System Packages](docs/packages.md)
* Desktop Environment and X config - TODO
* Security Considerations - TODO
* Default Background Services - TODO
* [Glossary of Terms and Codenames](docs/glossary.md)

### Guides

These are third-party guides I've found helpful in my explorations. Follow these at your own risk!

* [Running the Steam Deck’s OS in a virtual machine using QEMU](https://blogs.igalia.com/berto/2022/07/05/running-the-steam-decks-os-in-a-virtual-machine-using-qemu/)

## License

All text in this repository is licensed under the `LGPL-2.1+` in order to match the licensing of script files found in SteamOS 3. See [LICENSE](LICENSE) for more details. This repository may contain reproductions or snippets of SteamOS files licensed under `LGPL-2.1+`.

SteamOS 3 is governed by the terms of the `STEAM® END USER LICENSE AGREEMENT` found at the [SteamOS download page](https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=).
