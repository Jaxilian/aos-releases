# Installing AOS

For someone installing AOS on a machine of their own, from a release.
About twenty minutes, most of it the copy.

## Will it run here?

Before writing a stick, on the machine as it is now, under any Linux:

```sh
sh aos-check.sh
```

(br2ext/board/aos/aos-check.sh, nothing to
install.) It prints a verdict per item -- the CPU level, firmware and
Secure Boot, the GPU against the NVIDIA rule, which driver this Linux
uses for the Wi-Fi, sound and Bluetooth -- and writes
`~/aos-check-<host>-<date>.txt` to send with a bug report. A NO stops
AOS; a CHECK is something to look at in [known-issues.md](known-issues.md).

## What you need

- A 64-bit PC from about 2009 on (the CPU must support x86-64-v2, that
  is SSE4.2 and POPCNT: an Intel Core i3/i5/i7 or newer, an AMD from
  2011 on; a Core 2 or older will not boot it). Intel or AMD graphics work out of the box;
  NVIDIA cards from the RTX 20 series on use NVIDIA's open driver.
- A USB stick of 2 GB or more, for the installer. It is erased.
- A disk of at least 16 GB to install onto. **It is erased completely.**
  AOS does not share a disk with another system yet.
- UEFI firmware (legacy BIOS machines are not supported). Secure Boot
  may stay on: a release starts under it after one key enrolment at the
  first start (step 3; [secure-boot.md](secure-boot.md)).

## 1. Get the release and check it

From the release page, download `aos-<version>-x86_64.iso`, `SHA256SUMS`,
`SHA256SUMS.minisig` and the OS release key, `apm.pub` (key id
`CA2811A314D6EB0C`; the same file is `/usr/share/aos/os-keys/<id>.pub`
on every AOS machine). Then:

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
4. Pick the stick's **UEFI** entry. AOS starts with no menu (hold Space
   while it starts for the loader's entries). With Secure Boot on the
   first start shows shim's blue MokManager screen: any key, **Enroll
   key from disk**, the volume **AOS_ESP**, the file
   **ENROLL_THIS_KEY_IN_MOKMANAGER.cer**, Continue, **Yes**, Reboot;
   from then on everything signed with the AOS key starts. If the firmware does not list the stick at all, see
   [known-issues.md](known-issues.md) -- some refuse an ISO written to a
   stick.

The live desktop runs as `admin`, an account with no password: aos-rootd asks
for none there, and there is no lock screen, because whoever sits at a
live stick owns the machine.
Nothing on your disk is touched until you install. A stick made with
`usb.sh --release` installs too: it is a machine of its own, so the
installer asks for your password first.

If the screen goes black after the loader, restart, hold Space and pick
"AOS (live, safe graphics, no GPU driver)": it leaves the GPU driver out, which is the way in on hardware
whose driver misbehaves. Please report it ([known-issues.md](known-issues.md)
says how).

## 4. Install

Press the Super key (the one with the Windows logo) and pick **Install
AOS**:

1. **Licence**: the agreement (`/usr/share/aos/EULA.md`); I agree.
2. **Keyboard**: your layout, for the desktop and the console.
3. **Time zone**.
4. **Disk**: the disk to install onto. The stick you started from is never
   offered. Everything on the chosen disk is erased.
5. **Install**: "Also encrypt the whole disk", if you want it, locks
   programs, settings and logs too behind a passphrase asked for at
   every start, before the desktop; on a machine with a TPM 2.0 the
   disk opens by itself in that machine and the passphrase is kept as
   the recovery key. Then a progress bar, then
   **Restart**. Remove the stick when the machine turns off. If Secure
   Boot was off, the Done page also shows an eight-digit password: the
   next start's blue screen asks for it (Enroll MOK, Continue, Yes, the
   password, Reboot), so that Secure Boot can be turned on later. A missed screen is harmless.

The installed machine has no account yet. Its **first boot** asks the
keyboard and the zone again, then your user name and password: that
account owns the machine and is its administrator, root has no password
and logs in nowhere, and your home is encrypted with your password (opened
at every login, locked when you log out). It then shows a **recovery
key** once: write it down and keep it away from the computer. If you
forget your password, the key typed in its place opens your files;
without either they are gone, by design. A last page asks whether the
machine may send diagnostics reports: off unless you say yes
([privacy.md](privacy.md) says what one holds). **Continue** brings the
login screen.

The demo account of the live medium is never installed.

**A kiosk** -- a reception screen, a library terminal, a machine that
is one program -- is installed from the live medium's root console
(Ctrl+Alt+F2, which logs root in by itself on the live medium only):

```sh
aos-install --kiosk "firefox --kiosk https://example.org" /dev/sda
aos-install --kiosk "..." --user NAME --password-file FILE /dev/sda   # with an administrator's account
```

The machine then starts straight into that program, full screen, with
no shell or bar, as a password-less account that can do nothing else;
when the program ends it is started again within seconds. The command
is `/etc/aos/kiosk` on the installed machine, read at every start. The
program must be on the machine: AOS's own (`terminal`, `files`) are;
Firefox or Chrome are installed with `apm` as NAME first, from a tty
(Ctrl+Alt+F2). Without `--user` nobody can log in to the machine but
the kiosk session: root has no password.

On a machine installed with an owner, Settings -> Users -> Kiosk (or
`busctl call org.aos.Root /org/aos/Root org.aos.Root KioskOn s "CMD"`
after `aos-root unlock`) makes it a kiosk later, and `aos-kiosk off`
undoes it; the loader menu's (hold Space while the machine starts) "AOS
(no kiosk, login screen)" entry is the owner's way in meanwhile.

## 5. After the install

- **Wi-Fi (also a workplace's, with a user name), VPN (WireGuard),
  printers, sound, displays, Bluetooth, users**: the Settings
  application.
- **Svenska**: Settings -> Keyboard -> Language, at the next login. The
  shell and the lock screen stay in English for now.
- **Software**: Super, then Software -- AOS's own applications under
  Official, and Visual Studio Code, Firefox, Discord and Steam under
  Third-party once you switch that repository on (they are not part of
  AOS, and are marked).
- **Updates**: AOS looks once a day and says so when there are some.
  Software -> Update installs every application update and a new AOS in
  one step; a new AOS starts at the next restart. To go back to the one
  before if anything is wrong: hold Space at start for the loader menu
  and pick the older entry, or use the recovery screen's "Go back to the
  previous version of AOS" (it runs `aos-update --rollback`, which needs
  root). Software -> Updates -> "Follow the stable channel" takes only
  releases that have had their time on the newest one. A machine is updated, never reinstalled.
- **Encrypting later**: a machine installed without "Also encrypt the
  whole disk" is encrypted in place from the live stick, on mains
  power: `aos-encrypt DISK PASSFILE` on its root console, Ctrl+Alt+F2 (the passphrase on the file's
  first line), keeping everything on it.
- **Screenshots**: Print (the display), Shift+Print (the window); they
  are in Pictures/Screenshots.
- **Glass or not**: Settings -> Appearance. Glass everywhere is the
  default; Lightweight is opaque and easiest on a slow GPU.

## If something goes wrong

Settings -> About -> **Save Report** writes one file with everything a
bug report needs (the journal, the hardware, what is installed). Without a desktop, log in on the console and run `aos-report`.
