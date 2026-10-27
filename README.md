<!--
SPDX-FileCopyrightText: Copyright (c) 2026 Maulik Mistry
SPDX-License-Identifier: Apache-2.0
-->
# NuPhy Air75 V2 WebHID / VIA Linux Fix

This project provides a udev rule to resolve WebHID access permission issues (`NotAllowedError`) for the NuPhy Air75 V2 keyboard on Ubuntu and Debian-based Linux systems.

Copyright © 2026 Maulik Mistry

This project is licensed under Apache License 2.0. See the [LICENSE.txt](LICENSE.txt) file for details.

Please share support: 
- [PayPal](https://www.paypal.com/paypalme/m1st0)
- [Venmo](https://venmo.com/code?user_id=3319592654995456106&created=1753283702)

## Why this?

When attempting to configure a NuPhy Air75 V2 using VIA or the NuPhy Web Console on Linux using Chromium-based browsers, the WebHID API fails with a permission error.

Raw kernel `/dev/hidraw*` device nodes default to `root:root` (`0600`). Standard `TAG+="uaccess"` rules may fail on these interfaces if they are processed too late in the udev rule sequence.

**Lexical Ordering Failure:** Rules named `99-*.rules` evaluate *after* `/usr/lib/udev/rules.d/73-seat-late.rules`. Because `73-seat-late.rules` handles seat-related device processing and `uaccess` permissions, adding the `uaccess` tag in a later rule may be too late for the expected ACLs to be applied.

This solution provides a clean, device-specific udev configuration without resorting to world-writable `0666` modes or forced `plugdev` group assignments.

## Features

- Native `uaccess` implementation using dynamic POSIX ACLs.
- Eliminates security risks associated with world-writable (`0666`) permissions.
- Correct lexical ordering (`70-` prefix) to execute prior to `73-seat-late.rules`.

## Setup and Use

1. Clone this repository:

   ```bash
   git clone https://github.com/m1st0/nuphy_systemd_acl.git
   cd nuphy_systemd_acl
   ```

2. Create a symbolic link from the cloned repo to `/etc/udev/rules.d/`:

   ```bash
   sudo ln -s "$(pwd)/70-nuphy.rules" /etc/udev/rules.d/70-nuphy.rules`
   ```

3. Reload udev rules and re-trigger:

   ```bash
   sudo udevadm control --reload-rules
   sudo udevadm trigger
   ```

4. Unplug the keyboard USB cable and plug it back in (Wired mode).

## Verification

To verify that systemd-logind assigned dynamic user session access specifically to the NuPhy Air75 V2 nodes (without displaying unrelated system hidraw devices), run:

```bash
for dev in /dev/hidraw*; do
  if udevadm info --query=property "$dev" | grep -q "ID_VENDOR_ID=19f5" && \
     udevadm info --query=property "$dev" | grep -q "ID_MODEL_ID=3246"; then
    echo "== $dev (NuPhy Air75 V2)"
    getfacl -c "$dev"
  fi
done
```

Expected output showing active user session ACLs across the keyboard's device endpoints should be similar to:

```bash
== /dev/hidraw2 (NuPhy Air75 V2)
user::rw-
user:<your-username>:rw-
group::---
mask::rw-
other::---

== /dev/hidraw3 (NuPhy Air75 V2)
user::rw-
user:<your-username>:rw-
group::---
mask::rw-
other::---

== /dev/hidraw4 (NuPhy Air75 V2)
user::rw-
user:<your-username>:rw-
group::---
mask::rw-
other::---

== /dev/hidraw5 (NuPhy Air75 V2)
user::rw-
user:<your-username>:rw-
group::---
mask::rw-
other::---
```
