---
title: Upgrading
taxonomy:
    category: docs
---

This page describes how to upgrade one Mender Client version to a different Mender Client version.

## Minor and patch versions

<!--AUTOVERSION: "% -> %"/ignore-->
If you are upgrading from one minor version to another, such as 3.4.0 -> 3.5.0, or from one patch
version to another patch version, such as 3.5.1 -> 3.5.2, then no manual intervention is
needed. Minor and patch versions are always backwards compatible. This follows from Semantic
Versioning, which is described in more detail on [the Compatibility
page](../../../02.Overview/14.Compatibility/docs.md).

## Major versions

New major versions may come with changes that require manual migration steps. Changes of this sort
are sometimes necessary to fix problems that cannot be dealt with in minor releases, or enable new
behaviors that conflict with existing behaviors in some way.

<!--AUTOVERSION: "to % or later"/ignore-->
### Upgrade the Mender Client 3.x series to 4.0.0 or later

#### Overview

<!--AUTOVERSION: "In Mender Client %"/ignore-->
In Mender Client 4.0.0, the service was split into several smaller components. Previously the
`mender` binary tool handled everything; now there are several smaller ones with distinct roles:

* `mender-auth` for handling server communication and authentication
* `mender-setup` for handling Mender configuration setup
* `mender-snapshot` for handling snapshotting of live filesystems
* `mender-update` for handling updates.

Following that change, the Linux systemd service was also split from `mender-client` into
`mender-authd` and `mender-updated`.

<!--AUTOVERSION: "Mender Client % and later"/ignore-->
The following features and config are not available in Mender Client 4.0.0 and later:

- [Synchronized
  updates](https://docs.mender.io/3.6/overview/customize-the-update-process#synchronized-updates)
- [Update Management API](https://docs.mender.io/3.6/device-side-api/io.mender.update1)
- The
  [connectivity](https://docs.mender.io/3.6/client-installation/configuration-file/configuration-options#connectivity)
  options in the client setting. The new client does not cache connections, so these options are
  not needed anymore

<!--AUTOVERSION: "Mender Client % and later"/ignore-->
In Mender Client 4.0.0 and later the rootfs-image update type is no longer embedded in the client
codebase but is treated as any other external update module. A binary CLI tool `mender-flash` now
gets installed as part of the Mender Client to serve the needs of the external rootfs update module.

#### Upgrade using Debian packages

In the [Mender APT repositories](../../../10.Downloads/docs.md#install-using-the-apt-repository) `mender-client`
package corresponds to the Mender Client written in Go (version 3.x.y). This package is installed by default
from our [Express installation script](../../../10.Downloads/docs.md#express-installation). To upgrade from the
legacy Mender Client to the 4.x series on Debian and Ubuntu, you can install the `mender-client4` running:

<!--AUTOMATION: ignore -->
```bash
apt-get install mender-client4
```

This package will automatically install:

* a `mender-auth` package for server authentication:
  * provides the `mender-auth` binary
  * installs the `mender-authd` systemd service
* a `mender-update` package for doing updates:
  * provides the `mender-update` binary
  * installs the `mender-updated` systemd service
* the `mender-flash` tool
* the `mender-setup` tool
* the `mender-snapshot` tool

In addition, this package will also automatically remove:

* Legacy `mender-client` package:
  * removing the `mender` binary
  * disabling `mender-client` systemd service

After the upgrade, please update your scripts invoking the `mender` CLI interface to use the
correct binaries (`mender-auth` or `mender-update`).

For old Debian and Ubuntu distributions, if the Mender add-ons Debian packages (e.g.,
`mender-connect` and `mender-configure`) are already installed in the system, the upgrade
can fail with an error message similar to this:

<!--AUTOVERSION: "Conflicts: mender-client but 1:%-1+debian+buster is to be installed"/ignore-->
```
The following packages have unmet dependencies:
 mender-client4 : Conflicts: mender-client but 1:3.5.2-1+debian+buster is to be installed
E: Unable to correct problems, you have held broken packages.
```

In this case, ensure to upgrade the Mender add-ons Debian packages at the same time, for
example running:

<!--AUTOMATION: ignore -->
```bash
apt-get install mender-update mender-client4
```

#### Upgrade using Yocto

<!--AUTOVERSION: "Mender Client version % or later"/ignore "Yocto 5.0 (%)"/ignore "Yocto 4.0 (%)"/ignore-->
Mender Client version 4.0.0 or later will automatically be built on Yocto 5.0 (scarthgap) and
later. If you want to enable building of Mender Client version 4.0.0 or later while on
Yocto 4.0 (kirkstone) or older, please refer to [this specific section in our Troubleshoot
guide](../../../301.Troubleshoot/01.Yocto-project-build/docs.md#mender-client-version-4-0-0-have-been-released-but-my-build-still-uses-the-single-client-3-x-series).

#### Migration steps

Regardless of whether you are using Yocto or a Debian package, there are some migration steps which
are required. When referring to "scripts" below, we mean any Update Module, State Script, or other
script which executes on the device. All scripts that ship with Mender, and which you have not
modified, do not need to be changed, since they will be auto-updated by the package manager.

1. If any scripts call this command:
	* `mender bootstrap`
   Then they need to be changed to call `mender-auth bootstrap` instead of `mender bootstrap`.

2. If any scripts call this command:
	* `mender setup`
   Then they need to be changed to call `mender-setup` instead of `mender setup`.

<!--AUTOVERSION: "at least mender-artifact version %"/ignore-->
3. If any scripts call this command:
	* `mender snapshot`
   Then they need to be changed to call `mender-snapshot` instead of `mender snapshot`. Note that
   if you are using snapshotting through mender-artifact's "ssh" feature, then you need
   at least mender-artifact version 3.11.0.

4. If any scripts call one of these commands:
	* `mender check-update`
	* `mender commit`
	* `mender install`
	* `mender rollback`
	* `mender send-inventory`
	* `mender show-artifact`
	* `mender show-provides`
   Then they need to be changed to call `mender-update <COMMAND>` instead of `mender <COMMAND>`.

5. If any scripts call one of these commands:
	* `mender daemon`
   Then you will need several changes. Since `mender-auth` and `mender-update` are now separate
   daemons, you will need to start both of them, with `mender-auth daemon` and `mender-update
   daemon`, respectively.

6. If any scripts call either `journalctl` or `systemctl` with `mender-client` as an argument, you
   will need to change that. `mender-client` service has been replaced by two services
   `mender-authd` and `mender-updated`.
