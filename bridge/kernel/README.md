# Kernel patch: functionfs vendor descriptor passthrough

<!-- SPDX-License-Identifier: GPL-2.0-only -->

**This directory is GPL-2.0-only.** It is the one part of this repository that
is not MIT, and it cannot be relicensed: the patch modifies
`drivers/usb/gadget/function/f_fs.c`, making it a derivative work of Linux
kernel source. See [LICENSE](LICENSE) for the full text.

## What it does

`ffs_do_single_desc()` rejects any USB descriptor type it does not model, and
one rejection fails the **entire** descriptor set. That makes functionfs
unusable for any device whose interface must carry a vendor-defined descriptor.

For the Xbox 360 case specifically:

- type `0x21` (wired) lands in the `USB_TYPE_CLASS | 0x01` case and is rejected
  there, because the interface class is `0xFF` rather than HID, CCID or DFU
- type `0x22` (adapter) reaches `default` and is rejected

Without that descriptor, Windows' `xusb22` will not instantiate a controller.
The device enumerates and is never usable as a gamepad.

The patch accepts both types verbatim, scoped to vendor-specific interfaces, so
no standard class parsing behaviour changes.

## Building

`f_fs.c` needs only `u_fs.h`, `u_os_desc.h` and `configfs.h` from the tree.
`u_f.h` no longer exists; its contents moved to `include/linux/usb/func_utils.h`,
which ships in the headers package.

```sh
# fetch f_fs.c and its three headers for YOUR kernel version, then:
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo cp usb_f_fs.ko /lib/modules/$(uname -r)/updates/
sudo depmod -a
```

Installing to `updates/` makes it take precedence over the stock module at boot.

Developed against `6.12.75+rpt-rpi-v8`. Use source matching your own kernel.

## Warning

**A kernel update silently replaces this module with the stock one**, and
anything relying on the passthrough stops working with no error message.
Rebuild and re-run `depmod -a` after upgrading.

## Upstreaming

This is a reasonable upstream candidate. If submitted it goes as GPL-2.0 with a
`Signed-off-by` line under the Developer Certificate of Origin, which the patch
file already carries.
