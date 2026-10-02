# Installing AOS

For someone installing AOS on a machine of their own, from a release.
About twenty minutes, most of it the copy.

## What you need

- A 64-bit PC from about 2012 on (the CPU must support x86-64-v2; a Core
  2 or older will not boot it). Intel or AMD graphics work out of the box;
  NVIDIA cards from the RTX 20 series on use NVIDIA's open driver.
- A USB stick of 2 GB or more, for the installer. It is erased.
- A disk of at least 15 GB to install onto. **It is erased completely.**
  AOS does not share a disk with another system yet.
- Secure Boot turned **off** in the firmware settings. Nothing AOS ships
  is signed for it (see [policies.md](policies.md)).

## 1. Get the release and check it

From the release page, download `aos-<version>-x86_64.iso`, `SHA256SUMS`,
`SHA256SUMS.minisig` and the publisher's key, `apm.pub` (key id
`CA2811A314D6EB0C`). Then:

```sh
minisign -Vm SHA256SUMS -p apm.pub        # "Signature and comment signature verified"
sha256sum -c --ignore-missing SHA256SUMS  # "aos-<version>-x86_64.iso: OK"
```

If either fails, do not use the file.

## 2. Make the installer stick

On Linux, with the stick at `/dev/sdX` (check with `lsblk`; the wrong
letter erases the wrong disk):

```sh
sudo dd if=aos-<version>-x86_64.iso of=/dev/sdX bs=4M conv=fsync status=progress
```

On Windows or macOS, any tool that writes a raw image works (Rufus in
"DD image" mode, balenaEtcher).

## 3. Boot it

1. Power the machine fully off -- not a restart.
2. Put the stick in a USB port on the machine itself, not a hub.
3. Power on and open the one-time boot menu: Esc or F8 on ASUS, F12 on
   Dell and Lenovo, F9 on HP, Option on a Mac.
4. Pick the stick's **UEFI** entry. GRUB shows a menu; the first entry
   starts AOS. If the firmware does not list the stick at all, see
   [known-issues.md](known-issues.md) -- some refuse an ISO written to a
   stick.

The live desktop runs as a demo account, `admin`, password `123321`.
Nothing on your disk is touched until you install.

If the screen goes black after GRUB, restart and pick "AOS (live, safe
graphics, no GPU driver)": it leaves the GPU driver out, which is the way in on hardware
whose driver misbehaves. Please report it ([known-issues.md](known-issues.md)
says how).

## 4. Install

Press the Super key (the one with the Windows logo) and pick **Install
AOS**:

1. **Keyboard**: your layout, for the desktop and the console.
2. **Time zone**.
3. **Account**: your user name and password. This account owns the
   machine and is its administrator; the demo account is not installed,
   and root cannot log in on the console.
4. **Disk**: the disk to install onto. The stick you started from is never
   offered. Everything on the chosen disk is erased.
5. **Install**: a progress bar, then **Restart**. Remove the stick when the
   machine turns off.

## 5. After the install

- **Wi-Fi, sound, displays, Bluetooth, users, updates**: the Settings
  application.
- **Software**: Super, then Software -- AOS's own applications under
  Official, and Visual Studio Code, Firefox, Discord and Steam under
  Third-party once you switch that repository on (they are not part of
  AOS, and are marked).
- **Updates**: AOS looks once a day and says so when there are some.
  Software -> Updates installs every application update and a new AOS in
  one step; a new AOS starts at the next restart, and the boot menu's "AOS
  (previous version)" goes back to the one before if anything is wrong.
- **Screenshots**: Print (the display), Shift+Print (the window); they
  are in Pictures/Screenshots.
- **Glass or light**: Settings -> Display -> Appearance. Light is opaque
  and easiest on a slow GPU.

## If something goes wrong

Settings -> About -> **Save Report** writes one file with everything a
bug report needs (the journal, the hardware, what is installed). Without
a desktop, log in on the console and run `sudo aos-report`.
