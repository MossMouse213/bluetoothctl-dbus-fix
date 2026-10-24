# bluetoothctl: system D-Bus connection failure

This branch backports the BlueZ fix for a `bluetoothctl` SIGABRT when the
system bus cannot be reached. It targets BlueZ 5.87, including Arch's
`bluez-utils` 5.87-2 package.

## Symptoms and cause

Commands such as `bluetoothctl list` or `bluetoothctl show` may abort with:

```
dbus_connection_get_object_path_data: assertion "connection != NULL" failed
```

The client passed a failed connection from `g_dbus_setup_bus()` to the object
manager without checking it. Restricted execution environments, a missing bus
socket, or an invalid `DBUS_SYSTEM_BUS_ADDRESS` can expose the bug. An abort
with a different stack may have a different cause.

The backport preserves the connection error, reports it on stderr, cleans up,
and returns exit status 1. It does not grant bus access, start the Bluetooth
daemon, load firmware, or repair device pairing.

## Upstream status

The implementation is already upstream:

- Commit: https://github.com/bluez/bluez/commit/a57c6d52e7cc87e30768d09aadff4c7b3a736938
- Author: Gabriel Berfield
- Issue: https://github.com/bluez/bluez/issues/2446

Prefer a distribution package containing that commit when available. This
branch provides a backport and a standalone regression test; it does not
represent a new upstream fix. No duplicate bug report is needed.

## Build without replacing system packages

Install the distribution's development tools and headers for GCC, GNU Make,
Autoconf, Automake, Libtool, pkg-config, GLib, D-Bus, and Readline. Then, from
the source root:

```sh
./bootstrap
./configure --disable-systemd --disable-udev --disable-cups --disable-obex \
  --disable-manpages --disable-tools --disable-monitor
make -j4 client/bluetoothctl
python3 test/test-bluetoothctl-dbus ./client/bluetoothctl
```

Run the built client directly from a normal host terminal:

```sh
./client/bluetoothctl list
./client/bluetoothctl show
```

These commands query controller status. A powered-on controller does not
prove pairing or audio playback works. A sandbox that denies system D-Bus
access will still deny it after this fix; the client now exits cleanly and
explains why.

Avoid replacing `/usr/bin/bluetoothctl` with an unmanaged binary. This build
does not require `make install`, restarting Bluetooth, or changing system
configuration. Distribution maintainers can apply the implementation patch
to their package build instead.

## Validation on a 2019 Mac Pro

Validated on a MacPro7,1 running Omarchy, kernel
`7.2.11-arch1-Watanare-T2-1-t2`, BlueZ 5.87-2, D-Bus 1.16.2-2, and GLib
2.90.1. The active Bluetooth controller uses Realtek RTL8761BU firmware;
these results do not establish support for the Mac's internal controller.

- Original packaged client: reproduced the same abort under GDB using a
  missing bus socket; GDB intercepted SIGABRT.
- Patched client: missing socket with `list` and `show`, invalid address, and
  sandbox-denied access all produce a normal exit status of 1 and diagnostic.
- `--help` and `--version`: succeed without an accessible system bus.
- Normal host access: patched `list` and `show` both return 0 and find the
  powered-on adapter.
- Pairing, reconnect after suspend, and audio playback were not tested.

No MAC addresses, core dumps, or pairing keys are included in this report.

## Sharing

The backport credits the upstream author. The regression test and this guide
were prepared for MossMouse213, with assistance from GPT-6 via Codex.
The accompanying patch files can be applied to an unmodified BlueZ 5.87
checkout using `git am`. A Git bundle preserves the branch and its base for
offline transfer; neither format requires a GitHub account.
