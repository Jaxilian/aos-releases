# Security status

What the CVE report for a release says, and what was done about each
entry. The report itself is `aos-<version>-x86_64-pkg-stats.html` beside
every release (`release.sh` makes it with Buildroot's `pkg-stats`, against
NVD's feed). This page is its triage, redone when the report changes.

**Triaged 2026-10-01 against the 0.1.7 report: 64 CVEs in 19 packages.**

## Fixed by a version bump (in 0.1.8)

| Package | From | To | CVEs | Why it mattered |
|---|---|---|---|---|
| util-linux | 2.41.5 | 2.42.4 | CVE-2026-13595 | libblkid's partition probers run on every disk udev sees, so a crafted USB stick reached it |
| libxml2 | 2.15.3 | 2.15.4 | 9 (CVE-2026-86137..86144, 11979) | parsed by GTK, PipeWire, systemd and others |
| pcre2 | 10.47 | 10.49 | 6 (CVE-2026-89156..89162) | used by GLib, grep, systemd |
| gzip | 1.14 | 1.15 | CVE-2026-41991, 41992 | the LZH decoder on files a person opens |
| coreutils | 9.10 | 9.12 | CVE-2026-56391, 56392 | uniq -w and unexpand -t overflows |
| gawk | 5.4.0 | 5.4.1 | CVE-2026-40467..40469, 40553 | scripts on untrusted input |
| patch | 2.7.6 | 2.8 | CVE-2026-56288, 56289 | its seven Buildroot patches are upstream now and were dropped |

util-linux's three Buildroot patches were also upstream in 2.42 and were
dropped.

## Not applicable (ignored in `br2ext/external.mk`, with the reason there)

- **linux**: 13 of 14 are NVD records that match every kernel ever
  (CVE-1999-0656 is the ugidd RPC daemon, CVE-2007-4998 is cp), KVM host
  and nested-virtualisation bugs, and drivers AOS does not build (JFS,
  vmwgfx, NFC LLCP). CVE-2022-4543 (EntryBleed, a KASLR timing leak) has
  no software fix.
- **gcc** CVE-2023-4039: AArch64 only.
- **openssh**: OPIE (not built), RHEL 4/5's trojaned 2008 packages, and a
  Rowhammer theory against password authentication -- sshd on AOS takes
  keys only, and is off unless a key is installed.
- **glibc** CVE-2026-5435, 6238: the deprecated `ns_print*` debugging
  functions; nothing on the image calls them.
- **tar** CVE-2026-18477, 18508: incremental dumps and `--one-top-level`;
  `aos-update` uses neither, and extracts only signed tarballs.
- **python-setuptools** CVE-2026-59890: a build tool, not on the image.
- **terminal** CVE-2011-0189: Apple's Terminal. Our applications now carry
  an explicit vendor in their CPE, so a name match cannot happen again.

## Open, with the reason

| Package | CVEs | Exposure | Plan |
|---|---|---|---|
| linux 7.1.13 | CVE-2026-52972 (af_alg AEAD length) | local | the next 7.1.y point release, through the kernel package |
| binutils 2.45.1 | 8 (readelf/objdump, an XCOFF linker overflow) | local, a developer reading a hostile binary | binutils is the toolchain's version too: 2.47 rebuilds the toolchain, planned with the next toolchain bump |
| systemd 258.7 | CVE-2026-40223 (an assert, local) | needs a `Delegate=yes` unit with no `User=`; AOS ships none | systemd 260+ with the next Buildroot update |
| grub2 2.14 | CVE-2025-61662 (gettext use-after-free) | needs the GRUB prompt, i.e. the keyboard at boot | the next GRUB release |
| bison 3.8.2 | CVE-2026-56389, 56390 | a developer building a hostile grammar | no fixed release yet |

## How a fix reaches a machine

A package fix is a new AOS release: a version bump here, a tag, and
`release.sh --publish`. Every installed machine looks daily
(`aos-update-check.timer`), the desktop shows a notice, and Software ->
Updates (or `sudo apm upgrade`) writes the new release into the idle root
slot and boots it at the next restart, with the old one a menu entry away
(docs/upgrading.md). Applications from apm update the same way, in the
same step.
