# Making a USB stick that actually boots

This is the short version, then the long version, then everything that went
wrong the first time so it does not go wrong again. Every rule below is
there because it was broken once.

## The short version

```sh
./usb.sh            # build, pick the stick, install AOS onto it, verify it
```

Run it as yourself. It lists the sticks it can see and picks the only one
(or asks which), asks once for `YES`, asks once for your password, runs
`make`, refuses to go on if the image has an SSH key for root
(`AOS_DEV=1` overrides, for a development stick only), then boots
the live ISO in QEMU with the stick attached and types the install for you.
Ten minutes, most of it the build and the copy; the guest's transcript is
in `output/images/usb-install.txt`. `./usb.sh --no-build` skips the make,
`./usb.sh --test` boots the finished stick in QEMU afterwards as proof,
`./usb.sh --release` (the same as `--oobe`) makes a stick for a
person: no account until its first boot asks for the owner (see
"Accounts" below), `./usb.sh --seed=DIR` copies DIR into the demo
account's home afterwards (not with `--release`: that home is made at
the first boot), and `./usb.sh /dev/sdX` names the stick instead of
being asked.

`AOS_OUT=$PWD/output2 ./usb.sh ...` uses another Buildroot output
directory (a release built in output2 while output/ holds one not yet
published) for the build, the key check, the write and the log
(make-usb.sh passes it to write-usb.sh as root: sudo on a host that has
it, `ssh root@localhost` on an AOS host with a development build's key).

Then: stick into a **USB-A port on the machine itself**, machine fully **off**,
power on, open the **one-time boot menu** (Esc or F8 on ASUS, F12 on most
others), choose the **UEFI** entry for the stick.

That is the whole procedure. It is the only one that has booted on real
hardware here, and it produces a full, writable AOS install on the stick —
which is what a test machine needs: it takes updates like any install.

The same by hand, which is what `usb.sh` does for you and what the rest of
this page walks through:

```sh
make                                                  # the image must be current
sudo ./br2ext/board/aos/write-usb.sh                  # lists your stick's device name
sudo ./br2ext/board/aos/write-usb.sh /dev/sdX --install   # X = the letter it printed
```

In the QEMU window that opens: log in as `root`, then

```
aos-install --demo /dev/vda      # or without --demo: no account; the first boot asks for the owner
poweroff
```

(`--install --auto` is the middle ground: the same command, with
`auto-install.py` typing those two lines instead of you; it is what
`usb.sh` runs, with `--oobe` for `--release` and `--oobe`.)

**Do not `dd` the ISO to the stick expecting it to boot.** See "Why not the
live ISO" below.

## The long version, with what each step looks like

### 1. Build first

`write-usb.sh` writes `output/images/rootfs.iso9660`. If you changed
anything, run `make` first, or you install the previous image and wonder why
the fix is not there. `post-build.sh` prints one line at the end you should
read every time:

```
post-build.sh: no board/aos/authorized_keys -- sshd disabled
```

That is right for a stick: since 2026-10-07 no stick carries a root key,
and the person at the keyboard has the password. `installed 1 SSH
key(s) for root; sshd enabled` means a keyed build for the QEMU drivers
was left behind; `usb.sh` refuses it unless `AOS_DEV=1`. See
ssh.md.

### 2. Find the stick — and make sure the host actually sees it

```sh
sudo ./br2ext/board/aos/write-usb.sh
```

With no arguments it refuses and lists removable USB disks:

```
Removable USB disks currently attached:
  /dev/sda  29358 MiB   SanDisk 3.2Gen1
```

Rules from experience:

- **`sdX` is a placeholder.** Type the real letter. The script rejects
  `/dev/sdX` — that has been typed literally.
- **If it lists none, the stick is not attached**, whatever the physical
  situation looks like. `Problem opening /dev/sda ... The specified file does
  not exist` from any tool means the same thing. This stick has needed to be
  pulled out and pushed back in before the host would enumerate it, with no
  error anywhere. Re-seat it and run the listing again.
- The script only accepts a **removable, USB, whole disk**. It refuses the
  NVMe and it refuses partitions (`/dev/sda1`). That is deliberate: a typo
  here is your own filesystem.

### 3. Write it

```sh
sudo ./br2ext/board/aos/write-usb.sh /dev/sda --install
```

- It needs root; writing a block device does: `sudo` on a host that has
  it, `ssh root@localhost` on an AOS host running a development build
  with your key (an AOS release has no root at all). (`run-qemu.sh` must
  **not** run as root. Different tools, different rules.)
- It shows what it is about to erase and asks for `YES` — **capitals**.
  `yes` aborts. That has happened.
- It unmounts anything the desktop auto-mounted from the stick. You do not
  have to.

Then QEMU boots the live ISO with the stick attached as a disk. You will see
the loader hand over (no menu), the kernel log, and:

```
aos login:
```

### 4. Inside the VM: two commands, exactly these

```
root
aos-install --demo /dev/vda
```

Without `--demo` the installed stick has no account: its first boot
asks for the owner (see "Accounts" below).

- **It is `/dev/vda` inside the VM**, always. The stick is `/dev/sda` on the
  host and `/dev/vda` in the guest; the guest never sees a `/dev/sda`.
  Typing the host command (`/dev/sda --install`) into the guest gives
  `-bash: /dev/sda: No such file or directory`. That has happened too.
- It asks for `YES` again. Capitals.
- What follows, and how long:

  ```
  >>> Releasing /dev/vda
  >>> Wiping old partition tables and signatures from /dev/vda
  >>> Partitioning /dev/vda
  >>> Creating filesystems
  ```
  A `Cannot initialize conversion from codepage 850` message from
  `mkfs.fat` here is harmless — no locale tables on the image; it uses its
  built-in one.
  ```
  >>> Writing the core into slot a (/dev/vda3); this takes a while
                                                   the ~850 MB core, then read back through its verity
  >>> Mounting
  >>> The aos partition: var, the store, the homes
  >>> Creating swap file
      3924 MiB at aos/swapfile                     2-3 minutes, no progress shown
  >>> Installing bootloader                       shim, systemd-boot and loader.conf on the ESP (UEFI only)
  >>> Writing slot a's entry to the ESP           the slot's signed unified kernel image
  >>> Unmounting
  AOS installed on /dev/vda. Remove the install media and reboot.
  ```
  With an account there is a `>>> Account:` line before the bootloader.
  The swap step is silent and slow. It has not hung. Kernel lines like
  `clocksource: Watchdog remote CPU 1 read timed out` are QEMU noise.

- A `>>> note:` line after the bootloader about Microsoft's 2011 or 2023
  CA is information, not an error ([secure-boot.md](secure-boot.md),
  "The 2023 CA"). A `!!` line is an error.

Then:

```
poweroff
```

QEMU exits on its own. The script re-reads the stick and prints its layout;
what you want to see:

```
NAME    SIZE PARTLABEL PARTTYPENAME     FSTYPE LABEL
sda    28.7G
├─sda1    1M bios_grub BIOS boot                        (empty; kept so the slots keep their numbers)
├─sda2  512M ESP       EFI System       vfat   AOS_ESP
├─sda3    6G aos-a     Linux filesystem squashfs      (the core: squashfs + verity hash tree)
├─sda4    6G aos-b     Linux filesystem               (empty until the first update)
└─sda5 16.2G aos       Linux filesystem ext4   aos
>>> EFI system partition /dev/sda2 is FAT32    with:
bootx64  efi ...
Done. /dev/sda is ready.
```

If the top line shows `FSTYPE iso9660 LABEL ISOIMAGE` for the whole disk,
that is stale cache from an earlier live-ISO write: `sudo partx -u /dev/sda`
refreshes it. On an image with the wipe step it should not appear at all.

### 5. Boot the machine

In this order, because each has cost a reboot:

1. **Power the machine off completely.** Not a restart. Some firmware only
   enumerates USB at cold boot.
2. **Insert the stick before pressing power**, in a **USB-A port on the
   chassis** — not a hub, dock, or USB-C/Thunderbolt port, which many
   laptops cannot boot from at all.
3. Open the **one-time boot menu** at power-on, not the boot-order page in
   setup. The saved boot order lists configured entries; removable media
   appear in the one-time menu.
4. Pick the entry that names the stick, the **UEFI** one if the firmware
   offers both.
5. No menu: AOS starts. Hold Space while it starts for the loader's
   entries; `AOS (safe graphics, no GPU driver)` there is for when the
   screen goes black after boot — see below. With Secure Boot on the
   first start is MokManager's blue screen: enrol the AOS key from the
   stick ([secure-boot.md](secure-boot.md), "The user's one enrollment").

Firmware settings that must be right, checked once per machine:

- **Secure Boot on or off, both work** since 0.3.11: on, the first start
  enrols the AOS key (above). A keyless developer build (post-build.sh
  says UNSIGNED) needs it off.
- **Fast Boot off.** It skips USB enumeration on some boards.

### 6. If the stick is not in the boot menu at all

That happened five times here before it worked. The order to check:

1. Is it the `--install` layout, not a `dd` of the ISO? If it is the ISO,
   stop and use `--install`. See below.
2. Did the loader install (step 4: `>>> Installing bootloader` with no
   `!!` line after it)? If not, rebuild and reinstall.
3. Cold boot, chassis USB-A port, one-time menu (step 5). Try the other
   USB-A port.
4. Prove the stick on the host, without a reboot:

   ```sh
   sudo ./br2ext/board/aos/write-usb.sh /dev/sda --boot
   ```

   This boots the physical stick in QEMU behind the same UEFI firmware a PC
   uses, with the guest's writes diverted to a snapshot so the stick is not
   changed. A login prompt means the stick is good and the machine's
   firmware is the difference; no login prompt means the stick, and the
   serial output on your terminal says what broke. (`--boot --auto` makes
   the same check with no one watching and reports it in the exit status;
   `./usb.sh --test` runs it for you after installing.)

   Not `--test`: that mode writes the live ISO to the stick first and then
   boots it. It is the ISO's check, and on an installed stick it erases the
   install.

## Accounts: the demo one, and yours

The desktop session is a real user's, not root's. What you get in the
terminal is that user's shell, in that user's home. Nobody is root
(docs/no-root.md): an administrator's changes go through aos-rootd, which
asks that user's password.

**Demo (`usb.sh` with no account option, `aos-install --demo`).** The
live medium's account, `admin`, kept: a member of `wheel`, **no
password**, and aos-rootd asks for none (`/etc/aos/live`), so
there is no lock screen either: whoever holds the stick owns it. The
session starts as `admin` at boot (ade.service's autologin; the account
is in `/etc/systemd/system/ade.service.d/10-aos-user.conf`). Powering off,
rebooting and suspending need no password at all: `systemctl poweroff`
(or `shutdown now`, or `reboot`) typed in the terminal just does it,
because a polkit rule of ours lets the active seat session do those —
logind's own policy would demand an administrator's password through a
polkit agent the moment anyone else is logged in, over SSH say, and there
is no agent to ask. sshd refuses password authentication, so a stick
with the demo account on it is not a remote root hole. Root has no
password; the serial console logs root in by itself on this kind of
stick (`/etc/aos/live`), and nowhere else.

**Yours, or someone else's (`./usb.sh --release`, the same as
`--oobe`).** No account on the stick at all: `admin` and its
no-password administration are not installed. Its first boot comes up as a locked
`setup` user whose only program is the welcome: keyboard, time zone, a
name and a password, **Finish**. That makes the owner -- a systemd-homed
account (`wheel`, `pipewire`, `video`) whose home is a LUKS2 image
opened by the password at every login -- ends the serial console's
root autologin that the stick had until then (`/etc/aos/oobe`),
and shows the **recovery key** once; **Continue** restarts into the
login screen, which lists the owner, and closes the road: the helper
behind Finish refuses once an owner exists, and the `setup` account is
removed. Until then the stick has no password at all, so hand it over
rather than leave it lying around. Root stays reachable over SSH with
your key on a development stick (`AOS_DEV=1`), and aos-rootd covers
the rest.

## Why not the live ISO

Writing `rootfs.iso9660` to a stick with `dd` is the obvious thing, and it is
what every distribution's instructions say. It boots from a stick under
QEMU's UEFI. **On the ASUS ROG Zephyrus G14 used here (AMI Aptio, BIOS 305)
it has never once appeared in the boot menu**, across every fix that was
tried: a real GPT with the EFI system partition as a partition, a FAT16
ESP instead of the FAT12 Buildroot makes, the backup GPT moved to the end of
the stick, a 128-entry partition table. Same port, same stick, same firmware
settings under which a Fedora live stick boots. The cause was never found.

A `--install` stick is a plain GPT disk with a FAT32 EFI system partition at
the front — the same shape as the Fedora install on this laptop's own NVMe,
which the firmware boots every day. That is why it works where the ISO
does not, and why it is the documented route. The ISO remains right for
optical media and for QEMU. The mechanics of what the ISO needs to be
USB-bootable at all are in PLATFORM.md, "USB booting".

## After it boots: the three things that bit on real hardware

None of these show in QEMU. All three are fixed in the image now; the
entries here are so you recognise them if they come back in another form.

**Screen goes black a few seconds after the loader.** A GPU driver bound, evicted
the firmware framebuffer, and then failed — here because the `xe/` firmware
for the Intel Panther Lake iGPU was not on the image. Reboot, hold Space, pick
`AOS (safe graphics, no GPU driver)`, log in, and read the previous boot:

```sh
journalctl -b -1 -k --no-pager | grep -iE 'xe|i915|nvidia|drm|firmware'
```

Blind, with no console: the **power button once** is a clean poweroff,
**Ctrl+Alt+Del** a clean reboot. In a terminal, `systemctl poweroff` needs
no password (the polkit rule under "Accounts").

**Console floods with `PCIe Bus Error: severity=Correctable ... Timeout`
and shutdown seems stuck.** A PCIe device with no driver, not sleeping,
flapping its link tens of thousands of times a second — here the Realtek
card reader before `rtsx_pci` was added. If a new device does this, the
line names it (`device [10ec:525a]`); the fix is its driver, not silencing
the message. The screen entries now run at `loglevel=4`, so warnings go to
the journal and only errors reach the console.

**Touchpad dead, USB mouse fine.** Nothing is wrong with input handling;
the touchpad is not a USB device. It is an I2C-HID device behind the
chipset's I2C controller and the SoC's GPIO pinctrl, and with any of those
drivers missing it never appears (`ls /sys/bus/i2c/devices` is empty).
All of them are in the kernel now, see the "Input / laptop" section of
`linux.config.fragment`; if a new machine shows the same, `journalctl -k |
grep -iE 'i2c|hid|pinctrl'` says which link is missing.

**No network.** A USB-C Ethernet adapter gets DHCP with no setup. Wi-Fi
needs credentials once — the recipe is in ssh.md. If neither
interface exists at all (`ip link`), it is a missing driver, and it was
here: the Wi-Fi 7 chip needed `iwlmld`, the USB adapter needed the usbnet
drivers, neither of which the kernel had.

## Reinstalling

An installed stick takes new releases by an update (Software -> Update),
keeping its accounts and homes; that is the normal way since 0.3.3.
Running `--install` on it again is fine when a clean stick is wanted,
and erases everything on it. The installer scrubs the disk first:
`wipefs -a`, then the first and last 2 MiB zeroed. That step exists because
a stick that has been a Fedora ISO, then an AOS ISO, then two installs kept
the ISO's volume descriptor at sector 64 through all of it — `parted` and
`mkfs` never touch that sector — until GRUB refused the disk as having two
partition tables and the firmware stopped listing it. Two identities on one
disk; every tool that guesses which one it is looking at can guess wrong.

## The mistakes, in the order they were made

| What happened | The rule |
|---|---|
| ISO `dd`'d to the stick, not bootable: no partition table at all | The ISO is a hybrid now, but use `--install` for real machines |
| Hybrid ISO not listed: FAT12 ESP, backup GPT in the wrong place, 248 GPT entries — all fixed, still not listed | Stop fighting one firmware's ISO handling; a real GPT disk is what it boots |
| `sgdisk: /dev/sda does not exist` | The stick was not attached. Re-seat it; listing first |
| `/dev/sdX is not a block device` | X is a placeholder |
| `Type YES to continue: yes` → `Aborted.` | Capitals |
| `-bash: /dev/sda: No such file or directory` inside the VM | Inside the VM it is `aos-install [--demo] /dev/vda` |
| Booted, then black screen; `sudo shutdown now` blind did nothing | `xe/` firmware missing (the image then had no `sudo`); power button or Ctrl+Alt+Del |
| Console flooded with correctable PCIe errors | Card reader had no driver; every PCIe device needs one even if unused |
| Touchpad dead, USB mouse worked | The touchpad is I2C-HID: it needs the I2C controller, the LPSS and pinctrl drivers, and i2c-hid-acpi, all four |
| BIOS loader refused a reinstalled stick; firmware stopped listing it | Stale ISO signature at sector 64; the installer now scrubs first |
| Two fixes rebuilt, stick still had the old image | `make` before `write-usb.sh`, and read post-build's last line — or `./usb.sh`, which does both |
| Unattended install driven through QEMU's stdin on a pty died at GRUB | Drive the guest over a serial socket, as `boot-test.py` and `auto-install.py` do |
