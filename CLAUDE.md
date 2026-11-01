# CLAUDE.md

Fork of [pulpier/tidal-connect-hifiberry](https://github.com/pulpier/tidal-connect-hifiberry)
(upstream remote). It packages Tidal Connect for **HiFiBerry OS NG** (Debian 13 trixie,
arm64) as the `hifiberry-tidal-connect` `.deb`.

## Current work

Fixing the Android Tidal app crash caused by tdcn's incomplete mDNS advertisement
(upstream issue [#6](https://github.com/pulpier/tidal-connect-hifiberry/issues/6)).
**Read `docs/mdns-crash-investigation.md` first.** It has the root cause, the
evidence, the options considered, and the planned changes.

## What's in this repo (and what isn't)

- `vendor/bin/tdcn.gz`: the Tidal Connect daemon. **Proprietary, pre-built,
  stripped arm64 Rust binary. No source exists here or anywhere public.** Do not
  modify it or try to binary-patch it (license + impractical). The real bug lives in
  upstream's private `crates/tdcn/src/discovery.rs`.
- `vendor/bin/tdcn-ssl.gz`: certificate helper run by tdcn at startup (MIT, no
  source published).
- `vendor/tidal_connect_application.gz`, `vendor/certificate.dat.gz`: iFi's
  proprietary binary + ZenStream cert bundle, consumed only by tdcn-ssl. Must
  **not** be stripped (see `override_dh_strip` in `debian/rules`).
- `debian/`: all the actual packaging. `debian/rules` just gunzips and installs.
  `debian/tdcn-config.toml` → `/etc/tdcn/config.toml` (conffile, documents every
  key tdcn accepts). `debian/tdcn.service` → systemd **user** unit
  (`systemctl --user`).
- `.github/workflows/release.yml`: builds the `.deb` on a native arm64 runner
  when a tag is pushed. The tag must equal the `debian/changelog` version.

## Constraints

- tdcn runs as an unprivileged **systemd user service** (uid 1000 on the device).
  Anything that needs root at runtime (writing `/etc/avahi/services`, nft,
  `SocketBindDeny=`/`IPAddressDeny=`) is not available from the unit itself.
- Target is arm64 only. `dh_shlibdeps` needs real arm64 libs (`libasound.so.2`),
  so build the package on arm64 (the Pi, the GitHub arm64 runner, or an arm64
  container/VM). tdcn can't be run on an x86 dev machine except under qemu.
- Raspberry Pi 5 default kernel uses 16K pages. Don't introduce armhf/32-bit
  dependencies (they fail with "ELF load command alignment not page-aligned").
- Keep `debian/changelog` style: detailed prose paragraphs explaining *why*,
  matching upstream's entries.

## Build & test

```bash
# on arm64 (Debian trixie), from repo root
sudo apt install build-essential debhelper devscripts
dpkg-buildpackage -us -uc -b
sudo apt install ../hifiberry-tidal-connect_<ver>_arm64.deb
```

Test device: Raspberry Pi 5, hostname `hifimbr`, HiFiBerry OS NG. Useful checks:

```bash
systemctl --user status tdcn
journalctl _SYSTEMD_USER_UNIT=tdcn.service -f        # as root
avahi-browse -rpt _tidalconnect._tcp                  # what's advertised
ss -ulpn | grep 5353                                  # who holds mDNS sockets
```

Final verification needs the **Android** Tidal app's device picker. The macOS app
doesn't crash either way, so it's not a valid test.

## Conventions

- Work on branches. Don't push to `master` without asking.
- Upstream-facing text (issue comments, PR descriptions) should be factual and
  credit the reporters in #6.
