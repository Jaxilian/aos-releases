# AOS releases

Downloads of AOS, a Linux desktop operating system: one desktop, one
package manager, updates in A/B slots. Alpha software.

Each [release](../../releases) has:

- `aos-<version>-x86_64.iso` -- the installer (write it to a USB stick)
- `SHA256SUMS`, `SHA256SUMS.minisig`, `apm.pub` -- to check it
- `aos-<version>-x86_64-pkg-stats.html` -- the known-CVE report

Check before using:

```sh
minisign -Vm SHA256SUMS -p apm.pub        # key id CA2811A314D6EB0C
sha256sum -c --ignore-missing SHA256SUMS
```

Then: the install guide, `docs/install.md`, and what does not work yet,
`docs/known-issues.md` -- both copied below for each release. Installed
machines update themselves; this page is for installing.
