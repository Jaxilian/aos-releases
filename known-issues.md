# Known issues

What does not work yet, or works only partly, in the current alpha. Each
entry says what to do about it today. A problem that is not here: Settings
-> About -> Save Report, and send the file with a description to the
address in [../../SECURITY.md](SECURITY.md) (security problems) or as
an issue on the project.

## Installing

- **Secure Boot asks for one key enrolment.** The first start of a signed
  release with Secure Boot on shows shim's blue MokManager screen, where
  the AOS key is enrolled from the stick ([install.md](install.md) step
  3). Secured-core machines with Microsoft's third-party CA turned off
  refuse shim until that setting is on. Legacy BIOS machines are not
  supported since 2026-10-20: UEFI only.
- **Some firmware will not boot the ISO from a stick.** The image is a
  hybrid ISO; most UEFI firmware lists it, but some (seen: the AMI firmware
  of an ASUS ROG Zephyrus G14) does not. Workaround on a Linux machine:
  `./usb.sh` from the source tree installs AOS onto the stick as an
  ordinary system, which every firmware boots ([usb.md](usb.md)).
- **The whole disk is used.** No dual boot, no installing beside another
  system, no choosing partitions.
- **A forgotten password is the files gone.** Every home is encrypted
  with its account's password (systemd-homed); the recovery key shown
  once at the first boot is the only other way in. Programs, settings
  and logs are in the clear unless "Also encrypt the whole disk" was on
  at install (`aos-install --encrypt`), which adds a passphrase asked at
  every start, before the desktop ([policies.md](policies.md)). On a
  machine with a TPM 2.0 the disk opens by itself and the passphrase is
  the recovery key. A machine installed in the clear is encrypted later
  with `aos-encrypt` from the live medium, and `aos-tpm enroll` (or
  Settings -> About) adds the TPM key to an older encrypted one.
- **A password changed in Settings leaves the keyring on the old one.**
  homed changes the home's password, but the login keyring (oo7, where
  Chrome, VS Code and the like keep logins) is not re-keyed, so the next
  login cannot open it and those programs ask again.
- **A machine installed before 0.3 keeps its classic account**, with a
  plain home: it updates and works, and nothing converts the home.
  Reinstall for an encrypted one. Settings → Accounts can make new,
  encrypted accounts on it.
- **A home is not resized.** homed gives the owner most of the free
  space of the aos partition, sparse on the disk; what a second account
  made in Settings gets is the same default. Shrinking or growing one
  afterwards is homectl's job on a terminal.
- **The live session's account, `admin`, has no password**: aos-rootd asks for none, there is no lock screen, and the serial console is root's. It
  exists only on the installer; an installed machine has only the
  account you create, and root logs in nowhere there.

## Hardware

- **Two laptops have been tested**, an ASUS ROG Zephyrus G14 (2026,
  Intel Panther Lake, NVIDIA RTX 5070) and an ASUS Zenbook UM3406GA
  (AMD); docs/hardware.md has their pages. Everything else is untested;
  reports are what makes the compatibility list. Since 2026-10-29 only
  the G14 is available for rounds; the Zenbook's page stands as tested.
- **Sound** works on the G14's SoundWire (Intel's DSP with Cirrus
  amplifiers) and on HDA in testing.
- **The NVMe runs without its power saving** (APST off on the kernel
  line since 0.4.39): the G14's KIOXIA behind Intel VMD lost a
  completion every few hours and froze everything for 30 s. The cost
  is a little idle battery; a machine whose disk never stalled pays it
  too.
- **Bluetooth** works on the G14 (Intel BE201, on PCIe): Settings sees the
  adapter and the devices around. Pairing has no agent yet, so a keyboard
  that wants a passkey typed cannot pair.
- **The brightness slider** does nothing on the G14 (its backlight is
  dead under Fedora too).
- **A fingerprint** opens nothing since sudo went (0.4.31, docs/no-root.md):
  aos-rootd checks a password in the daemon, with no terminal of yours to
  ask a finger on; the lock screen
  and the login screen take the password (the login screen must: the
  password opens the encrypted home).
- **Suspend and lid close** work on the G14, and the screen locks before
  sleep. A lock screen that lost the keyboard after a resume (when the
  brightness display was up) is fixed in 0.2.8, not yet confirmed on the
  G14.
- **A port wired to the NVIDIA GPU** (the G14's HDMI) shows nothing: the
  desktop draws on one GPU and drives only its outputs. Planned, and big.
- **A display plugged back in** stayed black on 0.3.0; since 0.3.2 the
  desktop draws every display once after a hotplug. Not yet confirmed on
  hardware.
- **NVIDIA**: RTX 20 series and newer only (the open kernel modules); games
  through Proton have run on the Intel GPU, not yet on the NVIDIA one.

## Desktop

- **Glass shows only the wallpaper through a window**, never the windows
  behind it -- the cheap kind of blur, chosen so it runs on an integrated
  GPU. Settings -> Display -> Appearance -> Lightweight turns it off.
- **A new theme reaches an application when it next starts**; the desktop
  itself follows within seconds.
- **Switch user** (the power menu, 0.4.40) locks the screen and opens a
  second login on VT2; Ctrl+Alt+F1 and Ctrl+Alt+F2 move between the two.
  Two people at once, not more: one login screen per VT. The second
  session may come up behind its own lock screen; the password opens it.
- **No drag and drop between applications** yet. (Screenshots: Print for
  the display, Shift+Print for the window, Super+Shift+S without a Print
  key; they land in Pictures/Screenshots.)
- **Window controls**: the three lights at the top right of every
  window are the desktop's (0.3.2). A program that draws no title bar
  of its own (Blender, Spotify, X11 programs) gets a whole one from the
  desktop and resizes from a narrow band at its edges, with no resize
  cursor yet and no highlight when the pointer is over a light. One
  that draws its own bar whole (Firefox, Chrome, Steam) keeps it.
- **The shell and the lock screen are in English** when the system is
  set to Svenska (Settings -> Keyboard -> Language); every AOS
  application is translated.
- **The screen reader** (Orca with espeak-ng, Super+Alt+S or Settings)
  reads AOS's applications: each exposes its controls over AT-SPI
  through AccessKit. The shell (the bar, the launcher, the quick
  panel) and the lock screen do not yet, so they are silent to it; nor
  do third-party programs unless they bring their own support (GTK and
  Firefox do). No input methods (Chinese, Japanese, Korean typing): a
  big piece of work, not started.
- **Screen sharing** asks which display and shares it whole (no single
  window yet); each frame is read back through the CPU, so it costs
  more than it should.
- **X11 programs** need XWayland, a third-party package (Software ->
  AOS -> Compatibility); Steam installs it.

## Software

- **The app store's catalogue is the two indexes**: AOS's own
  applications, and as third-party Firefox, Chrome, Visual Studio Code,
  VSCodium, Zed, JetBrains Toolbox, Discord, Signal, Telegram, Slack,
  Zoom, Spotify, Steam, Lutris, Heroic, Prism Launcher, Blender, Krita,
  Inkscape, GIMP, Audacity, Kdenlive, qBittorrent, LibreOffice,
  ONLYOFFICE, Thunderbird, Node.js, Python, Java, .NET, podman,
  distrobox and docker. Not yet: OBS, HandBrake, Transmission.
- **Third-party packages are not part of AOS** and may break with an
  update of either; they are marked wherever they appear.
- **Firefox may show pages as boxes** on some machines (its sandbox and the
  fonts); fixed in testing, not yet confirmed on hardware.
- **Printing**: a plain text file sent with `lp` fails (cups-filters'
  texttopdf); what browsers and viewers print (PDF, PostScript) works.
  Scanners answer `scanimage -L`; there is no scanning program yet.
- **Firefox and Discord now live in a home of their own** (`~/.var/app/`,
  see [security-model.md](security-model.md)); a profile or login made
  by an earlier release stays in `~/.mozilla` or `~/.config/discord` and
  is not carried over. Downloads is shared, so what they saved there is
  where it was. A file dialog inside them shows their own home.

## Updates

- **A machine installed before 0.2.0** cannot update to 0.2.x: its
  updater knows only the old root slots ([layout.md](layout.md)).
  Reinstall it, once.
- **An update replaces the system, not a part of it**: about 850 MB per
  release (0.3.4's core). Delta updates are not planned before 1.0.
- **Two things still need a reinstall**: a home made before 0.3 (no
  conversion to an encrypted one) and a change of the partition layout
  (none planned: the slots have years of room). Since 0.3.3
  everything else, the loader included, comes with an update.
- **Every release reaches a machine as it is published** unless
  Software -> Update -> "Follow the stable channel" is on; the stable
  channel is filled by hand, after a release's time on edge, and is
  empty until the first promotion.
- **AOS's own applications show as installed in Software but do not
  upgrade one by one**: they are part of the release image, and a new
  version of one comes with the next system update.

## Security status

The CVE report of each release and what was done about every entry:
[security-status.md](security-status.md). What AOS defends against and
what it does not yet -- AOS's own applications are not sandboxed (third-party
ones are), the Secure Boot chain still rests on Fedora's shim
([secure-boot.md](secure-boot.md)), a machine installed before 0.3
has a classic home in the clear: [security-model.md](security-model.md).
