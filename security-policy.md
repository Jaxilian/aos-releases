# Security policy

Decided 2026-10-19 (docs/product-plan.md section 1). What the Adelon
Operating System promises about security fixes, and how a report
reaches us. docs/security-status.md is the current triage, generated
from `security/triage.csv`.

## Releases

- **Stable channel:** a release every month that carries every fix
  triaged since the last one.
- **Critical fixes** (reachable over the network, from a USB stick, or
  from a web page without the user doing anything) go out of band: the
  aim is a release within 7 days of the upstream fix.
- **The kernel** follows one stable series per stable release; a new
  series reaches the edge channel first.
- **Edge channel:** every release, as it is built.

## What is supported

A stable release receives fixes until the next stable release, plus 30
days. A machine updates itself (Software, `aos-update-check.timer`), so
staying supported means staying on. Hardware is supported as long as
its line in docs/hardware.md says "certified" or "works".

## The gate

No release is published with a CVE the triage has not seen: `release.sh`
runs Buildroot's pkg-stats against NVD and `security/triage.py check`,
and refuses when a reported CVE has no row in `security/triage.csv`,
when a row's package changed version since its triage, or when an
`open` row's promised `fixed_in` version is this release or older.

A fixed CVE that mattered gets an advisory `AOS-SA-YYYY-NNN` in the
release notes on github.com/Jaxilian/aos-releases: package, CVE, the
release that fixes it, whether a restart is needed.

## Reporting

- Address: **security@adelon.se** (to be set up with the company; until
  then, the repository owner on GitHub).
- We answer within 7 days, fix critical issues on the timeline above,
  and credit the reporter in the advisory unless asked not to.
- Disclosure: 90 days from the report, or the release that fixes it,
  whichever comes first.

## Keys

The Secure Boot signing key is kept on a hardware token (PKCS#11) with
two offline backups; the apm index key is a passphrase-protected
minisign key with two offline backups. docs/secure-boot.md says what a
leaked key costs and how it is rotated.
