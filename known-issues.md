# Known issues

What does not work yet, or works only partly, in the current alpha. Each
entry says what to do about it today. A problem that is not here: Settings
-> About -> Save Report, and send the file with a description to the
address in [../../SECURITY.md](../../SECURITY.md) (security problems) or as
an issue on the project.

## Installing

- **Secure Boot must be off.** Nothing is signed for it; a machine with it
  on will not start the installer. Decided, not planned
  ([policies.md](policies.md)).
- **Some firmware will not boot the ISO from a stick.** The image is a
  hybrid ISO; most UEFI firmware lists it, but some (seen: the AMI firmware
  of an ASUS ROG Zephyrus G14) does not. Workaround on a Linux machine:
  `./usb.sh` from the source tree installs AOS onto the stick as an
  ordinary system, which every firmware boots ([usb.md](usb.md)).
- **The whole disk is used.** No dual boot, no installing beside another
  system, no choosing partitions.
- **No disk encryption.** `/home` on LUKS is planned; until then anyone
  with the disk can read it ([policies.md](policies.md)).
- **The live session's account, `admin`, has no password**: sudo asks for
  none, and there is no lock screen. It exists only on
  the installer; an installed machine has only the account you create.

## Hardware

- **One laptop has been tested**, an ASUS ROG Zephyrus G14 (2026, Intel
  Panther Lake, NVIDIA RTX 5070). Everything else is untested; reports are
  what makes the compatibility list.
- **Sound on SoundWire laptops** (Intel's DSP with Cirrus amplifiers, as
  on the G14) is found but silent so far. HDA sound (most desktops and
  older laptops) works in testing.
- **Bluetooth** on the G14 (Intel BE201, on PCIe): the adapter comes up,
  the Settings page does not see it yet.
- **Suspend and lid close** have not been tested on hardware.
- **NVIDIA**: RTX 20 series and newer only (the open kernel modules); games
  through Proton have run on the Intel GPU, not yet on the NVIDIA one.

## Desktop

- **Glass shows only the wallpaper through a window**, never the windows
  behind it -- the cheap kind of blur, chosen so it runs on an integrated
  GPU. Settings -> Display -> Appearance -> Light turns it off.
- **A new theme reaches an application when it next starts**; the desktop
  itself follows within seconds.
- **No drag and drop between applications** yet. (Screenshots: Print for
  the display, Shift+Print for the window, Super+Shift+S without a Print
  key; they land in Pictures/Screenshots.)
- **X11 programs** need XWayland, a third-party package (Software ->
  AOS -> Compatibility); Steam installs it.

## Software

- **The app store's catalogue is small**: AOS's own applications, and
  Visual Studio Code, Firefox, Discord and Steam as third-party.
- **Third-party packages are not part of AOS** and may break with an
  update of either; they are marked wherever they appear.
- **Firefox may show pages as boxes** on some machines (its sandbox and the
  fonts); fixed in testing, not yet confirmed on hardware.
- **Firefox and Discord now live in a home of their own** (`~/.var/app/`,
  see [security-model.md](security-model.md)); a profile or login made
  by an earlier release stays in `~/.mozilla` or `~/.config/discord` and
  is not carried over. Downloads is shared, so what they saved there is
  where it was. A file dialog inside them shows their own home.

## Updates

- **A machine installed before 0.1.0** (one root partition) cannot update
  itself; reinstall it.
- **An update replaces the system, not a part of it**: about 650 MB per
  release. Delta updates are not planned before 1.0.
- **AOS's own applications show as installed in Software but do not
  upgrade one by one**: they are part of the release image, and a new
  version of one comes with the next system update.

## Security status

The CVE report of each release and what was done about every entry:
[security-status.md](security-status.md). What AOS defends against and
what it does not yet -- third-party programs are not sandboxed, the root
is not verified, the disk is not encrypted: [security-model.md](security-model.md).
