# Security policy

This is the policy for AOS, the distribution built from this tree. For the
Buildroot build system underneath it, see upstream:
https://buildroot.org/ and security@buildroot.org.

## Status

AOS is alpha software from one maintainer. The promised cadence (a
monthly stable release, critical fixes within 7 days of the upstream fix,
a release supported until the next one plus 30 days) and the CVE gate
every release passes are in
[br2ext/docs/security-policy.md](security-policy.md); the
current triage is
[br2ext/docs/security-status.md](security-status.md). Until
the company exists, the effort behind the promise is one person's.

## Reporting a vulnerability

Email jax.carls@protonmail.com with "AOS security" in the subject
(security@adelon.se once the company is set up). Do not
open a public issue for anything that could be exploited.

You will get an acknowledgement, a judgement of whether it is a
vulnerability in AOS or in an upstream package, and a fix or a mitigation
in a release when one exists. Please allow 90 days before publishing;
coordinated disclosure earlier is fine once a fix ships.

## What is AOS's, and what is upstream's

AOS assembles upstream software — the kernel, glibc, systemd, Mesa, OpenSSL
and the rest — and adds its own: ade, apm, the applications, the
configuration in `br2ext/`, and the installer. A vulnerability in AOS's own
code is fixed here. A vulnerability in an upstream package is reported
upstream, and AOS picks up the fix by updating the package.

## How a fix reaches you

A new release. An installed machine looks for one two minutes after a
boot and daily, and Software -> Update (or `apm upgrade`) writes it
into the idle root slot, signed and checked, with the old one a rollback
away (Settings -> Advanced, or the hidden loader menu) ([br2ext/docs/upgrading.md](upgrading.md)).
Applications from apm update in the same step. Since 0.3.3 a machine is
updated, not reinstalled. "Follow the stable channel" in Software waits
for a release to be promoted after its time on edge.

## Known limitations

- Secure Boot works through Fedora's Microsoft-signed shim with the AOS
  key enrolled once in the MOK list. Until a shim of our own is signed,
  someone with the disk can put an older signed release back, and
  anything Fedora signed boots too
  ([br2ext/docs/secure-boot.md](secure-boot.md)). With
  Secure Boot off the TPM unlock protects a disk taken out of its
  machine, not a machine whose boot path was swapped.
- The cores (the OS itself) are not encrypted: they hold nothing
  private, and dm-verity covers their integrity. Every home is encrypted
  (systemd-homed); the rest of the aos partition is encrypted only when
  chosen at install, or later with `aos-encrypt` from the live medium.
- AOS's own applications are not sandboxed yet; third-party ones are.
- The live ISO's account, `admin`, has no password and aos-rootd asks for none: whoever sits at the stick owns the machine. It is never
  installed; the installer makes the owner at the first boot
  (`aos-install --demo` keeps it, for test sticks only).

The full model, and what is still to build:
[br2ext/docs/security-model.md](security-model.md).
