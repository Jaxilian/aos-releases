# Policies: what AOS promises, and what it has decided

Short statements an evaluator asks for. Each one is a decision, written
down so it is not re-argued; the roadmap items they close are named.

## Releases and support

A 0.x release monthly, tagged, with the ISO, its signed `SHA256SUMS` and
the legal-info manifest published together
(publishing.md). Each release is supported until the
next one: fixes go into the next monthly release, not into the old one.
There are no point releases before 1.0 unless a fix cannot wait a month,
in which case the release is simply cut early. A release is not made from
a `-dirty` tree (release.sh refuses).

Every release goes to the **edge** channel as it is published; **stable**
gets a release promoted from edge after its time there with no fault
found, without a rebuild (`release.sh --promote stable`). A machine
follows stable with `CHANNEL=stable` in `/etc/aos/update.conf`, which
Software's "Follow the stable channel" writes
([upgrading.md](upgrading.md)).

An installed machine moves to a new release by a base-OS update (below).
Since 0.3.3 a machine installed once is never reinstalled for a release;
only a change to the partition layout or a pre-0.3 (pre-homed) home
needs one. A machine on 0.1.x cannot update to 0.2.x and is reinstalled
once ([layout.md](layout.md)).

## Security reporting

[SECURITY.md](SECURITY.md) at the repository root: the address, what
happens to a report, the 90-day disclosure window, and what is AOS's to
fix versus upstream's.

## No telemetry

AOS sends nothing anywhere on its own. No usage reports, no crash reports,
no update pings beyond the package index fetch that `apm update` makes
when you or the Settings application ask for one, and no identifier that
would let a server tell one machine from another. The diagnostics bundle
that `aos-report` writes stays on the machine unless a person sends it:
Settings' "Send a report", or the first-boot consent toggle, off by
default ([privacy.md](privacy.md) lists everything that leaves a machine). This is true of the image today and it is a promise for every
release: a change that broke it would be a bug, and it would be reverted.

Three things on the image talk to a server on their own, and none of
them sends anything about the machine: the daily update check (below)
fetches the signed indexes, systemd-timesyncd asks a time server for the
time, and cups-browsed with avahi listens for printers on the local
network while publishing nothing. fwupd fetches the LVFS's firmware list
only when someone runs `fwupdmgr`, and its automatic reports are off.

Third-party software does what it does; Firefox, Discord, Steam and Visual
Studio Code have their own telemetry and their own settings for it. That is
part of what "third-party" means on AOS.

## Network: closed by default

An AOS machine answers nothing it was not asked. The firewall
(`/etc/nftables.conf`, loaded before the network) drops every inbound
packet that is not a reply, ICMP, a DHCP answer or SSH, plus the ports an
installed package declared in its recipe (`[sandbox] ports`,
`vpn_ports`; apm writes them to `/opt/apm/exports/firewall/`, Software
shows them on the package's page, Settings -> Network switches them,
porting.md); sshd itself runs
only when the person turns remote login on in Settings -> Network (a
developer's image built with an SSH key has it on; `release.sh` refuses
to publish one); systemd-resolved's LLMNR
and mDNS responders are off, and avahi answers no name query. The kernel
has Landlock, Yama and lockdown, which need no policy; there is no
AppArmor or SELinux policy -- a known gap, not a decision
([security-model.md](security-model.md)). auditd runs; usbguard is
installed and off until a policy is written for the machine. Updates are looked for
daily (`aos-update-check.timer`, which refreshes the signed indexes and
asks whether a new AOS is out) and the desktop says when there are some;
nothing is installed without the person: `apm upgrade`, Software ->
Updates or Settings does that. The CVE report of each release and its
triage are in [security-status.md](security-status.md).

## Secure Boot: on, after one key enrolment

Decided 2026-10-14 (0.3.11); the loader changed 2026-10-20. A signed
build starts with Secure Boot on: Fedora's Microsoft-signed shim, then
systemd-boot and each slot's unified kernel image (kernel, microcode,
initramfs, the command line with the verity root hash) signed with the
AOS key, which the owner enrols once in shim's MOK list at the first
start of the stick, or which the installer queues with a one-time
password when Secure Boot was off. Every kernel module is signed,
NVIDIA's included, and refused unsigned under Secure Boot. With Secure
Boot on the TPM unlock (below) binds the disk to that chain; with it off
it protects a disk taken out of its machine only. What is still open (a
shim of our own, downgrade refusal, lockdown) and how the keys are kept
are in [secure-boot.md](secure-boot.md) and product-plan.md section 4.
Installs are UEFI only; legacy BIOS boot went with GRUB.

## Encryption: every home, always; the whole partition by choice

Decided 2026-10-07 (0.3.0). Every account is a systemd-homed one: its
home is a LUKS2 volume of its own, unlocked by its password at login
and locked when the last session ends, so a stolen laptop is a stolen
laptop and nothing more, with no choice to make at install. A recovery
key is made with the home and shown once at the first boot; a forgotten
password without it is the data gone, by design, and the setup says so.
The owner is therefore made at the machine's first boot, by the
installed system itself, for every kind of install: the installer asks
no account. Machines installed before 0.3 keep their classic account
(reinstall to get an encrypted home).

The installer's "Also encrypt the whole disk" (`aos-install --encrypt`)
puts LUKS2 around the aos partition as well: programs, settings, the
journal. On a machine with a TPM a second key slot is sealed to it (PCR
7) and the disk opens by itself at boot; the passphrase is the recovery
key, asked on the console before the desktop when the TPM refuses (no
TPM, its state wiped, the disk in another machine) or when there is no
TPM at all. `aos-tpm` adds or removes the TPM key later (Settings ->
About does the same), and `aos-encrypt` from the live medium encrypts an
install made in the clear, in place. The escrow of the recovery key
waits for a management server. The swap file exists only on the aos
partition. The cores
stay in the clear: they are the public release images, and verity
covers their integrity ([layout.md](layout.md)).

## Base OS updates

A release's core image (a squashfs with its dm-verity hash tree), named
in a `SHA256SUMS` signed with the apm key, written by `aos-update` into
the idle one of two core slots and checked through its verity; systemd-boot gives it three tries and the desktop coming up confirms it, else the
previous slot boots, and Settings -> Advanced ("start the other version"),
`aos-update --rollback` or the hidden loader menu (hold Space) goes back by
hand ([upgrading.md](upgrading.md)). Each slot's unified kernel image
carries its own kernel arguments and profiles, and once a release is
confirmed `boot-sync` brings the loader files on the ESP up to the
release's. Accounts,
`/home`, installed programs and settings live on the aos partition, which
no update touches; the kernel comes with the core, not from apm. Application updates go through `apm upgrade`; both are on the
Software application's Update page. A disk installed before 0.2.0
is reinstalled once, which erases it.
