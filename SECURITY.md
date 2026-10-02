# Security policy

This is the policy for AOS, the distribution built from this tree. For the
Buildroot build system underneath it, see upstream:
https://buildroot.org/ and security@buildroot.org.

## Status

AOS is alpha software. Security fixes are best effort, from one maintainer,
with no service level. That is the honest state of things, and it will be
updated here when it changes.

## Reporting a vulnerability

Email jax.carls@protonmail.com with "AOS security" in the subject. Do not
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

Today: a new release, installed from a new image. A base-OS update
mechanism is on the roadmap (`br2ext/docs/roadmap.md`, Phase 1); until it
exists, that is the answer, and it is the reason to prefer applications from
apm, which do update in place.

## Known limitations

- Secure Boot is unsupported; the kernel and modules are unsigned.
- The root filesystem cannot be encrypted; there is no initramfs.
- The default build carries a demo account. Published images must be made
  with `aos-install --release` (see `br2ext/docs/publishing.md`).
