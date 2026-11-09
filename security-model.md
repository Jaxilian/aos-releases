# The security model

What AOS defends its users against, how, and what is still to build.
This is the page the word "secure" is measured against; the CVE report
of each release is [security-status.md](security-status.md), the
promises are [policies.md](policies.md).

## The values

1. **AOS protects its users' privacy and integrity.** Nothing leaves the
   machine that the person did not send. No telemetry, no accounts, no
   identifiers ([policies.md](policies.md)).
2. **AOS protects its users from threats, outside and inside.** From the
   network, from a stolen laptop, and from a program they installed
   themselves.
3. **AOS provides an easy way to do everything.** Protection that needs a
   terminal, a policy file or a judgment call has failed; the safe way is
   the default and the only one on offer.
4. **AOS strives for "it just works".** A protection that breaks a
   program people need is a protection they will switch off.

The person owns the computer. AOS serves them, not a vendor: there is no
one else to report to, and no way in for anyone they did not let in.

## Who AOS is defending

- **Someone's grandmother**, who installs what a web page tells her to,
  types her password where she is asked, and never opens Settings.
- **A child** who downloads "free RAM", a cracked game, or a Discord
  plugin, and runs it. The program is the attacker.
- **A household**, where one account's documents are not another's.
- **An enterprise**, which asks how the machine is updated, what runs
  on it with what rights, what happens when a laptop is stolen, and
  wants every answer in writing and checkable.

The threats, in the order the defences below address them:

| Threat | What it looks like |
|---|---|
| A malicious program | Installed by the person, runs as them, reads their files, keys and sessions, talks to the network, stays after a restart |
| The network | A port that answers, a service that can be asked to do something, a download that was tampered with on the way |
| A stolen machine | The disk read in another computer; the OS altered and put back |
| A compromised AOS | Our own build machine, signing key or repository, or someone claiming to be it |
| Another account on the same machine | Reading or changing what is not theirs |

Not defended against, and said so: an attacker with physical access and
time on a machine with Secure Boot off (with it on, the signed chain of
[secure-boot.md](secure-boot.md) holds, with the gaps that page lists),
a compromised CPU or firmware,
and the person themselves typing their password into a program they
should not have trusted -- the sandbox limits what that program then
gets.

## What is in place

**Updates and software come signed, or not at all.** apm refuses an
index that no trusted key verifies; the image ships the official key.
`aos-update` writes a release's core image into the idle slot only
when `SHA256SUMS` carries the signature of an OS release key, from the
core's own `/usr/share/aos/os-keys` that apm never reads, so the index
key cannot sign an update nor the OS key an index; it reads it back through
its verity hash tree, and the loader boots the new slot on trial (three
tries) before the desktop confirms it ([upgrading.md](upgrading.md)).

**The cores cannot be changed.** A root slot is a squashfs image under
dm-verity: every block the kernel reads is checked against a hash tree
whose root hash the slot's signed unified kernel image carries, so a
block altered on the disk
is an I/O error and a core written half way does not mount
([layout.md](layout.md)). What is changed in the system's programs does
not outlive a restart, and a disk altered in another computer does not
boot that slot; the signing keys the store and the updates trust are
the core's, bound read-only over the aos partition's copy, so nothing
written there can add one. (The /etc overlay, /var and the store are
writable by root, so a bug that reaches root can leave something there
for the next start: see "What stays open".)
A tampered download, mirror or publisher is a refused download.

**The network is closed.** nftables drops everything inbound that is
not a reply, except DHCP, ICMP and port 22. Nothing listens on 22
unless the person turns remote login on in Settings -> Network, with a
key and never a password; while it is on, the bar shows **SSH open** in
red in the middle of the screen, so a door opened for a reason is not
forgotten. A published image carries nobody's key: `release.sh` refuses
an image with a root key on it (ssh.md).

**A stolen disk can be made unreadable.** The installer's "Encrypt the
disk" puts LUKS2 around the aos partition -- homes, programs, settings,
logs -- unlocked by the account's password at every start, before the
desktop ([layout.md](layout.md)). It is a choice at install, off by
default, and one passphrase covers every account on the machine. On a
machine with a TPM 2.0 the installer also seals a key to it, so the
disk opens by itself in its own machine and the passphrase becomes the
recovery key; with Secure Boot off that protects a disk taken out of
the machine, not one whose boot path was swapped, and with it on only
the signed chain unseals it ([secure-boot.md](secure-boot.md)). Stored logins -- what Chrome, VS Code and the like
keep through libsecret -- live in the Secret Service's keyring in the
home, encrypted with the login password that PAM hands it at login,
so they are as safe as the home and nothing asks a second password.

**The screen is shown only to the screen-sharing portal.** The
compositor offers screen capture (wlr-screencopy) to one program, the
portal's backend at /usr/libexec/xdg-desktop-portal-wlr, checked by
the executable and its inode; every other program, sandboxed or not, is
refused. The portal asks which display before anything is shared.

**USB devices can be held to a list.** usbguard is in the image and off:
an organisation writes the list (`usbguard generate-policy`) and turns
it on. The audit daemon records what the kernel's audit rules ask for.

**Programs cannot read each other's memory, or the kernel's.** Yama
limits ptrace to a process's own children; kernel pointers and the
kernel log are root's; BPF and perf counters are root's; the running
kernel cannot be replaced from userspace (`/etc/sysctl.d/10-aos-hardening.conf`).
The kernel builds with the memory-corruption checks Fedora's does and the
security modules a desktop can use without a policy -- Landlock, Yama,
lockdown -- and not SELinux, Smack, TOMOYO or AppArmor, which do
nothing without a policy nobody has written and would imply one.

**Everything is built with the usual hardening**: PIE, full RELRO,
`-fstack-protector-strong` and `_FORTIFY_SOURCE=2`, as Fedora and
Ubuntu do.

**Accounts are separate, and each home is encrypted.** Every account
is a systemd-homed one: its home a LUKS2 volume opened by its own
password at login and locked when its last session ends, so one
account's files are closed to another's and to whoever takes the disk;
a recovery key shown once at the first boot is the only other way in
([layout.md](layout.md)). The first account is an administrator and the
rest are not. Root has no password in any image (`root:*`), su, pkexec
and polkit's agent helper carry no setuid bit, and polkit has no
administrators (`/etc/polkit-1/rules.d/30-aos-admin.rules`): nothing
asks for an administrator's password and gets one, and the actions that
hand wheel a root operation without one (gvfs' admin backend, fwupd's
firmware flash, bolt's enrolment, logind's choice of what boots next)
are refused for everyone. There is no sudo. What an administrator may
do to the machine is aos-rootd's (no-root.md): one root
daemon on the system bus with one method per change, each checking its
own arguments -- there is no "run this command" -- for a member of wheel
who gave their own password in the last fifteen minutes; every call is
a line in the journal, and a sandboxed program cannot reach it. A USB stick mounts `nosuid,nodev`, so a
root-owned setuid shell on one is a file. An owned machine starts
at a login screen (greetd and ade-greeter) and the session locks before
sleep; a machine installed for someone else asks for its owner at the
first boot. The live system's
account has no password either: nothing on the ISO lets anyone in by a
password that every copy shares, and whoever sits at the stick owns the
machine, so aos-rootd asks it for none, there is no lock screen there, and
its serial console logs root in by itself -- as does a machine's before
its owner exists, and nobody's afterwards.

**A repository is the unit of trust, and only AOS's own may run as
root.** Official software comes from AOS's repository, checked against
the key in the core. Anyone else -- a vendor, a developer releasing to
themselves -- publishes a repository of their own, signed with their own
key: Software's "Add a repository" fetches that key from the address,
checks that it signs the index there and pins it, so nothing but that
key is trusted for it, and an older signed index is refused as a
replay. Each such repository is a page of its own in Software, labelled
as not part of AOS. Its packages must be prebuilt and carry no system
service and no install hooks: nothing from it ever runs as root, and
its programs run in their sandbox like any other. AOS's third-party
repository (compatibility software) is labelled the same way and is
never a dependency of anything official.

**A program starts with the session only if the person lets it.** A
package may ask to (a VPN client, a sync tool), like it asks for the
camera; its page in Software shows the switch "Starts when you log in",
and the person's choice wins. The desktop starts such programs once per
login through their launchers, so through their sandbox, as a click
would; removing the program removes that too.

**A program sees what its package declares, and the store says so.** A
package's manifest carries a `[sandbox]` section -- the files and
folders of the home it may share (`Downloads`, `Projects`,
`.gitconfig:ro`), SSH, the network, the camera, the microphone, the
developer tools, the discrete GPU, X11 -- and apm writes the program's
command and desktop entry to run through `aos-sandbox` with exactly
those flags: bubblewrap, unprivileged, the OS read-only, the package and
what it depends on, the session's sockets, and a home of its own,
`~/AppData/<org>.<name>` (visible in Files; deleting it resets the
program), that holds nothing of the real home but what it was given.
There is no whole-home mode any more (2026-10-23): `home = "full"`, which
installed manifests still say, runs as private with a warning. The
store's package page shows "Sees: its own files and Downloads" before
Install, and once installed the page's Permissions section shows what it
may see and lets the person change it. A program gets no camera and no
microphone unless its package declares them or the person switches them
on; sound still plays. The discrete GPU (`gpu = true`, `;gpu`, `--gpu`)
lifts the Mesa-only driver lists inside the sandbox and sets NVIDIA
render offload, as `prime-run` does: games declare it, everything else
stays on the integrated GPU so the discrete one can sleep. X11
(`x11 = true`, `;x11`, `--x11`) gives the program the socket of the
session's XWayland, the one `DISPLAY` names and only when it is the
person's own (`/tmp/.X11-unix/X<n>`), and `DISPLAY`: every X11 client can
read every other X11 window's keys and contents, so only a package that
draws through X11 declares it (Steam and its games, Android Studio,
Zoom); without it the socket is not there and DISPLAY is unset. Until
0.4.63 that held for the path socket only: XWayland also listened on
the abstract `@/tmp/.X11-unix/X<n>`, which needs no file and is reached
from anything in the host's network namespace, so a sandbox without
`--x11` reached the X server with `DISPLAY=:0` all the same, and so did
gamescope's own XWayland inside Steam's sandbox. Since 0.4.63 ade's
XWayland and gamescope's open the path socket only, and every sandbox
runs in a Landlock domain scoped against abstract unix sockets made
outside it (`/usr/libexec/aos-landlock`, Landlock ABI 6): a program
reaches the abstract sockets its own sandbox made and no others.
XWayland asks for no cookie; it serves its own uid and refuses
another account's. A program
started from the desktop (`--launch`) keeps its sandbox, and with it
everything the sandbox was given -- the microphone, the camera, the
folders, the D-Bus proxy -- while any process of it lives that is not
a known helper (a session bus, crashpad, gpg's and ssh's agents): so
an app that hands off to a new copy of itself (Godot's editor,
Sublime Text) keeps running, and closing a window does not by itself
end the sandbox while something it started is still running; stopping
its scope does (since round 132; before, the sandbox ended with its first
process). Every other run, from a terminal or a hook, ends when its
program exits. A program
that listens on the network declares its ports the same way (`ports`,
`vpn_ports` for WireGuard tunnels only, 1024 and up): the page says
"Listens on the network" before Install, apm opens exactly those in the
firewall on the system store and closes them with Remove, Settings ->
Network switches them off and on, and a rule file that does not load
is set aside at boot rather than run without a firewall
(porting.md). An open port exposes that program's bugs
to whoever can reach it; its sandbox is what contains them.

The person's say is `~/.config/aos/sandbox/<org>.<name>`, outside every
program's view: line 1 the flags over the package's (empty: the
package's own), then one grant per line, `rw PATH` or `ro PATH`, relative
to the home. aos-sandbox and aos-sys (`sandbox::check`) apply the same
rules to every share and grant, and a line that breaks them is left out:
never the home itself, nothing under `AppData`, `.var`, `.config`,
`.local` or `.cache` (other programs' data and shared settings), never
`.ssh` or `.gnupg`, a hidden file or folder read-only only, and a path
that resolves (through a symlink) somewhere that breaks them is refused
too. SSH is a flag of its own (`;ssh`, `--ssh`): `~/.ssh` read-only,
`known_hosts` writable over it, and the session's ssh-agent socket
(`ssh-agent.service`; `ssh-add` once in the Terminal unlocks a key for
both). Without it the program has no keys and no agent. What a running
program was given is written at its start to
`$XDG_RUNTIME_DIR/app/<org>.<name>/view`, for ade-portal.

Single files outside its folders reach a program through the document
portal: `$XDG_RUNTIME_DIR/doc` inside is the portal's folder for that
program alone. The portal's D-Bus interface is not on the program's bus
proxy: the portal takes every `app-aos-*` unit for a host program, which
may read and change every document of every program (closed 2026-10-23;
GTK's drag-and-drop of files into a sandboxed program stops with it). A
file named on a desktop entry's command line (Open With in Files, a
double click) takes the same way: each argument of a `--launch` start
that is an existing absolute path or `file://` URI outside the
program's view is handed to `ade-portal --doc <org>/<name> PATH`, and
the program gets the path inside `$XDG_RUNTIME_DIR/doc` that comes back;
one the portal refuses, or that is a link or not its own real path, is
left out with a line in the journal, never passed unchanged; a folder
is passed as it is (2026-11-07). The person's theme,
`~/.config/aos/theme`, is bound read-only at the same place in every
sandbox, so AOS's own apps keep its colours. A declared share that does not exist yet
is made, read-only ones too, as long as it passes the rules above.
AOS's own apps keep their data in their AppData folder like any other
program: at the first sandboxed start of aos/calendar, weather, clock,
notepad, calculator and videos, their old folders under `~/.config`,
`~/.local` and `~/.cache` are moved there, once, never over a folder
already there; the Clock's alarms reach the person's systemd through
`aos-clock.path` and `/usr/libexec/aos-clock-sync`, which reads
`clock --plan`, accepts only `alarm <id> <OnCalendar>` and
`timer <unix-seconds>` lines and writes the timers itself, since the
program cannot reach systemd from its sandbox. A
package that declares nothing runs as the person, and the store says
"everything, undeclared".

A drive's SMART (`/run/aos/disks/<drive>.json`, written hourly by the
root timer aos-disks.timer through smartctl) is readable by every
account, for the Disks app: health, temperature, hours, the serial
number; sysfs shows the same serial to everyone already. Nothing an
account writes reaches a drive through it (2026-10-27).

## What stays open

Root is nobody's: no account can become root, and aos-rootd does one
named thing at a time for an administrator who gives their password.
The /etc overlay, /var and /opt/apm are writable by root, so a bug that
reaches root -- through aos-rootd, a root service or the kernel -- can
leave something behind for the next start; the cores cannot be changed
and the trusted keys are read from the core, and that is the extent of
it. With Secure Boot off, anyone at the keyboard can edit the boot
line, boot another medium, and read or change an unencrypted aos
partition; with it on, the TPM key is bound to PCR 7 only, so a signed
AOS live medium unseals an installed disk until the key is bound to the
kernel image too (signed PCR 11 policies). For the releases after the
one that removed sudo, the previous root slot still holds it and can be
chosen at boot by whoever holds the keyboard; it goes when an update
overwrites that slot. The kernel's surface for a local account is a
desktop's: user namespaces (the sandbox needs them), /dev/kvm for the
seat's active account, /dev/net/tun and /dev/fuse for everyone; /proc is
mounted `hidepid=invisible`, so an account sees its own processes only
(polkitd, in the `proc` group, still sees every caller's; libsystemd's
cgroup lookups are patched to take the host's root when PID 1 is
hidden, br2ext/patches/systemd/0002). Official
package hooks run as a restricted root under systemd-run when apm
installs system-wide. The full list, with what would close each:
open-weaknesses.md.

## What is next, in order

1. **No root for anyone** (no-root.md): phase 1 shipped
   in 0.4.30 and phase 2, aos-rootd in place of sudo, in 0.4.31 (above); the trust root in
   the core, a password on the recovery screen's rollback, and your own
   repositories follow. What stays weak afterwards is kept honest in
   open-weaknesses.md.
2. **The rest of the sandbox.** The private home and the declared
   directories are in (above), and so is the file dialog the desktop
   draws (ade-portal's chooser): a single file picked outside the
   program's view reaches it through the document portal and nothing
   else, and a folder it lacks is asked for (Allow / Don't allow),
   granted and the program restarted into it. AOS's preinstalled apps
   run in it too, each with a `[sandbox]` of its own. What is still to
   come: Landlock fencing the view a second time from inside, and a
   package that cannot be installed without a declaration, once every
   recipe has one. The declaration is what answers the child with the download: a
   "free RAM" package that asks for the whole home says so on its page.
2. ~~**Verified root slots.**~~ Done 2026-10-06 (0.2.0): the cores, above.
   The root hash is on the ESP, named by the signed `SHA256SUMS` of the
   release that wrote it; a signed root hash checked by the firmware is
   Secure Boot's job, which is after stable.
3. ~~**The disk encrypted.**~~ Done 2026-10-06 (0.2.2): LUKS2 around the
   aos partition, a choice at install. ~~**A home per account**~~: done
   2026-10-07 (0.3.0), systemd-homed, every account its own LUKS volume
   unlocked by its own password at login, with a recovery key shown once
   ([policies.md](policies.md)). The cores stay in the clear: there is
   nothing private in them, and verity covers their integrity.
4. ~~**Secure Boot**~~: done 2026-10-14 (0.3.11) and 2026-10-20
   ([secure-boot.md](secure-boot.md)): Fedora's Microsoft-signed shim,
   systemd-boot and a unified kernel image per slot signed with the AOS
   key, enrolled once in the MOK list; every module signed and enforced
   under Secure Boot. Still open there: lockdown (it refuses
   hibernation), and a shim of our own so that a lower SBAT generation
   is refused and Fedora's certificate is no longer trusted
   (product-impl.md R4).
5. **An outside measure.** The CIS benchmark for a Linux desktop run
   once against a release and the differences explained here, so the
   claims above are checked by someone else's list.

## How a report reaches us

[../../SECURITY.md](SECURITY.md): the address, the window, and
what an alpha promises.
