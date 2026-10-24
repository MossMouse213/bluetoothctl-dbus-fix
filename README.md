# bluetoothctl D-Bus crash fix for BlueZ 5.87

[![Bluetoothctl regression](https://github.com/MossMouse213/bluetoothctl-dbus-fix/actions/workflows/regression.yml/badge.svg)](https://github.com/MossMouse213/bluetoothctl-dbus-fix/actions/workflows/regression.yml)

A tested backport and regression suite for `bluetoothctl` aborting with
`dbus_connection_get_object_path_data` and `connection != NULL` when system
D-Bus is unavailable. Useful on Omarchy/Arch and in restricted environments.

**The implementation is already fixed upstream.** This repository helps users
of older packages reproduce, test, and backport it. It is not a new Bluetooth
hardware driver or an official Omarchy/BlueZ project.

## What changes

Before: a failed system-bus connection causes SIGABRT and may create a core dump.
After: the original connection error is printed and the client exits with status 1.

The backport preserves the original author's credit:

- [Upstream fix by Gabriel Berfield](https://github.com/bluez/bluez/commit/a57c6d52e7cc87e30768d09aadff4c7b3a736938)
- [Existing upstream report](https://github.com/bluez/bluez/issues/2446)

Prefer your distribution's package containing that fix when available. Do not
apply this patch to a version that already includes it or open a duplicate
upstream issue for the same crash.

## Build and test

Requirements: Git, Python 3, GCC, Make, Autoconf, Automake, Libtool, pkg-config,
and development headers for GLib, D-Bus, and Readline.

From this repository's root, use a fresh directory called `source`:

```sh
git clone --depth 1 --branch 5.87 https://github.com/bluez/bluez.git source
git -C source am "$PWD"/patches/*.patch
cd source
./bootstrap
./configure --disable-systemd --disable-udev --disable-cups --disable-obex \
  --disable-manpages --disable-tools --disable-monitor
make -j4 client/bluetoothctl
python3 test/test-bluetoothctl-dbus ./client/bluetoothctl
./client/bluetoothctl list
./client/bluetoothctl show
```

`git am` needs your Git committer name and email configured. The standalone
regression test is also included at this repository's root.

The failure tests need no Bluetooth hardware and do not change controller
settings. A normal host terminal is needed for successful system-bus queries.
Do not use `sudo` to bypass a sandbox restriction; use the intended host context.

## What was verified

On a 2019 Mac Pro (MacPro7,1) with Omarchy and BlueZ 5.87-2:

- Missing system-bus socket: `list` and `show` exit 1 without aborting.
- Invalid bus address and sandbox-denied access: clean failure with a diagnostic.
- Help and version: work without a bus.
- Normal host access: `list` and `show` succeed and find a powered-on controller.
- Both patches apply to a clean 5.87 base; the resulting source tree matches
  the tested checkout.

See [validation and hardware details](VALIDATION.md). The active controller uses
Realtek RTL8761BU firmware. This is not proof of support for the Mac's internal
Bluetooth controller. Pairing, audio, suspend/resume, and reconnection have not
been tested.

[GitHub Actions](https://github.com/MossMouse213/bluetoothctl-dbus-fix/actions/workflows/regression.yml)
repeats the build and failure tests on pushes and pull requests. The badge above
shows the latest hosted result; local tests have passed.

## Installation and troubleshooting

Run the built binary directly first. This project does not overwrite
`/usr/bin/bluetoothctl`, change system settings, or restart services.
Distribution maintainers can use the implementation patch in their package.

This fix does not grant D-Bus access, load firmware, start bluetoothd, or fix
pairing. If the fixed client prints a connection error, resolve the reported
bus problem in the environment that ran it. If it sees a powered-on adapter
but a peripheral fails to connect, that is a separate issue to investigate.

## Sharing and contributing

The `patches/` directory contains two email-format Git patches. An optional
Git bundle is available locally for offline transfer but is not tracked here.
The code is GPL licensed; see [COPYING](COPYING) and file-level SPDX notices.

When reporting a new issue, include the exact command, package version,
controller model, and relevant diagnostic output. Redact MAC addresses and
personal paths; do not attach raw core dumps or pairing keys.

Prepared by MossMouse213, with assistance from GPT-6 via Codex.
