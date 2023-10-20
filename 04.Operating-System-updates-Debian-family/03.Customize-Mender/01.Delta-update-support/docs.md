---
title: Delta update support
taxonomy:
    category: docs
    label: tutorial
---


If you are using [Mender Professional](https://mender.io/product/features?target=_blank) or [Mender
Enterprise](https://mender.io/product/features?target=_blank), you have access to robust delta updates. In this section we describe how to enable support for delta updates on your devices,  by installing the `mender-binary-delta` Update Module with your Yocto Project build.

Once your devices support installing delta updates, see [Create a Delta update Artifact](../../../06.Artifact-creation/05.Create-a-Delta-update-Artifact/docs.md) for a tutorial on how to create a delta update from two Operating System updates.

## Prerequisites

In order to use delta update, you must be using a read-only root filesystem. There are several ways of achiving this and it is out of the scope of this tutorial. Generally speaking, just make sure the bootloader and the `/etc/fstab` don't contain any `rw` reference.

For additional troubleshooting, you can check the [Delta updates troubleshoot guide](../../../301.Troubleshoot/03.Mender-Client/docs.md#delta-updates).

## Download `mender-binary-delta`

If you are using *hosted Mender*, download the `mender-binary-delta` archive with the following
command:

<!--AUTOVERSION: "mender-binary-delta/%/mender-binary-delta-%.tar"/mender-binary-delta-->
```bash
HOSTED_MENDER_EMAIL="myusername@example.com"
curl -u $HOSTED_MENDER_EMAIL -O https://downloads.customer.mender.io/content/hosted/mender-binary-delta/1.4.1/mender-binary-delta-1.4.1.tar.xz
```

Replace the value of `HOSTED_MENDER_EMAIL` with the email address you used to sign up on *hosted Mender*, then enter your hosted Mender password when prompted to proceed.
**NOTE**: if you signed up using your Google or GitHub login, use the email address linked to that account and enter `x` as the password.

On the other hand, if you are using *on-premise Mender Enterprise*, download using the following
command:

<!--AUTOVERSION: "mender-binary-delta/%/mender-binary-delta-%.tar"/mender-binary-delta-->
```bash
MENDER_ENTERPRISE_USER=<your.user>
curl -u $MENDER_ENTERPRISE_USER -O https://downloads.customer.mender.io/content/on-prem/mender-binary-delta/1.4.1/mender-binary-delta-1.4.1.tar.xz
```

## Get the relevant files

<!--AUTOVERSION: "mender-binary-delta-%.tar.xz"/mender-binary-delta-->
The archive `mender-binary-delta-1.4.1.tar.xz` contains the binaries needed to generate and apply deltas.

<!--AUTOVERSION: "mender-binary-delta-%.tar.xz"/mender-binary-delta-->
Unpack the `mender-binary-delta-1.4.1.tar.xz` in your home directory:

<!--AUTOVERSION: "mender-binary-delta-%.tar.xz"/mender-binary-delta-->
```bash
tar xvf mender-binary-delta-1.4.1.tar.xz
```

The file structure should look like this:

```
├── aarch64
│   ├── mender-binary-delta
│   └── mender-binary-delta-generator
├── arm
│   ├── mender-binary-delta
│   └── mender-binary-delta-generator
├── licenses
│   └── ...
└── x86_64
    ├── mender-binary-delta
    └── mender-binary-delta-generator
```

### The `mender-binary-delta-generator`

Copy the generator compatible with your workstation architecture to `/usr/bin`, for a `x86_64` one, it should look like this:

<!--AUTOVERSION: "mender-binary-delta-%"/mender-binary-delta-->
```
sudo cp mender-binary-delta-1.4.1/x86_64/mender-binary-delta-generator /usr/bin
```

### Integrate `mender-binary-delta` into your image

Run the following steps as part of the Golden Image modifications in your device, they should be executed before copying the golden image into your workstation. If you are following the [recommended workflow](../../02.Convert-a-Mender-Debian-image/docs.md#recommended-workflow), execute these commands as part of step 2 "make modifications" and before step 3 "power off the device".

<!--AUTOVERSION: "mender-binary-delta-%.tar.xz"/mender-binary-delta-->
Download and extract the `mender-binary-delta-1.4.1.tar.xz` just like in the [previous section](#get-the-relevant-files).

Delta updates require the `mender-binary-delta` Update Module, so copy it into `/usr/share/mender/modules/v3/`.

Assuming a `x86_64` device, then use the following command, modify it accordingly to your device's architecture:

```
sudo cp mender-binary-delta-1.4.1/x86_64/mender-binary-delta /usr/share/mender/modules/v3/
```

Then, create a file named `mender-binary-delta.conf` inside `/etc/mender/`

The mender-binary-delta.conf content should look like this:

```
{
  "RootfsPartA": "/dev/mmcblk0p2",
  "RootfsPartB": "/dev/mmcblk0p3"
}
```

But change `mmcblk0p2` to the right location of the A partition and instead of `mmcblk0p3`, the location of the partition used as B.

## Next steps

For information on how to create delta update Artifacts, see [Create a Delta update Artifact](../../../06.Artifact-creation/05.Create-a-Delta-update-Artifact/docs.md).

For more information about delta updates, including how to deploy them, as well as troubleshooting, see the
[Mender Hub page about `mender-binary-delta`](https://hub.mender.io/t/robust-delta-update-rootfs/1144?target=_blank).
