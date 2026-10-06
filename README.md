# Udev Rules

Gives USB devices fixed names like `/dev/ydlidar`, so programs find the right device even when Linux numbers them differently on each boot.

Linux names USB-serial devices `ttyUSB0`, `ttyUSB1`, ... in enumeration order, which depends on probe timing and can change between boots.
A udev rule fixes this with an "if → then": if a device looks like the lidar, then create `/dev/ydlidar` pointing at whichever `ttyUSBX` it got this boot.
Programs open `/dev/ydlidar` and never need to know the number.

This repo is a guide for writing udev rules.
The files in `rules/` are worked examples for the AgileX LIMO robot.
They shouldn't be copied as-is, because device IDs and USB ports differ between machines.

## How udev rules work

When a device appears, udev checks it against every rule in `/etc/udev/rules.d`.
Each rule is one line of comma-separated keys:

```text
KERNEL=="ttyUSB*", KERNELS=="*-3*", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", ATTRS{devpath}=="3", MODE:="0666", GROUP:="dialout", SYMLINK+="ydlidar"
```

Match keys are the "if" part and assignment keys are the "then" part.
If every match key matches, udev applies the assignment keys.

| Key | Kind | Meaning |
| --- | --- | --- |
| `KERNEL` | `==` (match) | Kernel name of the device itself (`ttyUSB0`) |
| `KERNELS` | `==` (match) | Kernel name of the device or any parent (`1-3`) |
| `ATTRS{...}` | `==` (match) | sysfs attribute of the device or any parent |
| `MODE`, `GROUP` | `:=` (assign and lock) | Device node permissions and group, locked so later rules can't change them |
| `SYMLINK` | `+=` (append) | Adds a persistent symlink in `/dev`, keeping any added by other rules |

A plain `=` assigns without locking.
All `KERNELS`, `ATTRS` and `SUBSYSTEMS` keys in one rule must match the same parent device.

### File naming

Rule files are named `<number>-<name>.rules`, like `99-ydlidar.rules`.
udev reads every rule file in filename order, and the number at the front of the filename controls where a file falls in that order.
Files with higher numbers are read later, so a `99-` file runs after the system's own rules and can build on what they set.

### Device node permissions

`/dev/ttyUSB0` is a file, and like any file it has an owner, a group and read/write permissions.
By default it's owned by `root:dialout` with mode `0660`, so only root and members of the `dialout` group can open it.
`MODE:="0666"` lets any user read and write it, so programs like ROS nodes can open the device without `sudo`.

## Write a rule

### 1. Find the device

Watch the kernel log, then plug the device in:

```bash
sudo dmesg -w
```

The kernel logs a line like `cp210x converter now attached to ttyUSB0`.

### 2. Read its attributes

```bash
udevadm info -a -n /dev/ttyUSB0 | grep -E 'looking at|KERNELS|idVendor|idProduct|devpath|serial'
```

`udevadm info -a` walks from the device up through its parents and prints each one's attributes.
Each `looking at parent device` line starts a new block.
All match keys in a rule should come from a single block.

### 3. Pick match keys

`ATTRS{idVendor}` and `ATTRS{idProduct}` identify the kind of device.
If they're enough to tell the devices apart, nothing else is needed.

If two devices share the same vendor and product IDs, add one of these:

- `ATTRS{serial}` if each device has its own serial number, so the name follows the device wherever it's plugged in
- `ATTRS{devpath}` and `KERNELS` if the serials are the same, so the name follows the physical USB port instead

With port-based matching, a device moved to a different port loses its symlink.

### 4. Write the `.rules` file

Create a file named like `99-<device>.rules` with one rule per line, using the example above as a template.

## Test and apply

Copy the rule in and check that it matches:

```bash
sudo cp 99-<device>.rules /etc/udev/rules.d/
udevadm test "$(udevadm info -q path -n /dev/ttyUSB0)" 2>&1 | grep -E 'LINK|MODE|GROUP'
```

`udevadm test` replays the device through every rule and prints each match with its file and line number, like `99-ydlidar.rules:1 LINK 'ydlidar'`.
It doesn't need `sudo` to show matches.
On systemd older than 256 it also tries to create the symlinks, so running it without `sudo` prints harmless "Permission denied" lines.

Then reload udev and check the symlink:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -l /dev/<symlink>
```

Reloading applies the rules without a reboot.
If the symlink doesn't show up, replug the device.

## Examples (LIMO)

| File | Symlink | Device | Matches on |
| --- | --- | --- | --- |
| `rules/99-ydlidar.rules` | `/dev/ydlidar` | YDLIDAR | USB port `3` |
| `rules/99-limo-usb.rules` | `/dev/ttylimo` | LIMO base controller | USB port `7.4` |

Both devices use a CP2102 USB-serial chip (`10c4:ea60`), and both chips report the same serial number `0001`.
Vendor ID, product ID and serial can't tell them apart, so these rules match the physical USB port with `devpath`.

The LIMO image ships older copies named `ydlidar.rules` and `limo-usb.rules`.
When installing these examples, remove the old copies so the rules don't apply twice:

```bash
sudo rm /etc/udev/rules.d/ydlidar.rules /etc/udev/rules.d/limo-usb.rules
```

## Docs

- `man udev` for the full rule syntax
- `man udevadm` for the `info`, `test` and `control` commands
