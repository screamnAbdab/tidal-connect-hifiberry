# Investigation: Android Tidal app crashes on tdcn's mDNS advertisement

Status: root cause known, interim workaround verified on hardware, permanent fix
not yet implemented. Upstream report: [pulpier/tidal-connect-hifiberry#6][issue6].

Investigated 2026-10-31 – 2026-11-01 on a Raspberry Pi 5 running HiFiBerry OS NG
(Debian 13 trixie, kernel `6.18.50+rpt-rpi-2712`, 16K pages) with
`hifiberry-tidal-connect` 2.0.5.

---

## 1. Symptom

With the package installed and `tdcn` running, opening the cast/device picker
(top-right icon on the player screen) in the **Android** Tidal app crashes the app
immediately. Disabling Tidal Connect on the HiFiBerry makes the crash go away.
The macOS Tidal app sees and uses the device fine.

The crash is a `NullPointerException` in Tidal's own code while it builds the
device list (adb logcat, from issue #6):

```
java.lang.NullPointerException: Attempt to invoke virtual method
  'java.lang.Class java.lang.Object.getClass()' on a null object reference
    at q2.e.<init>(SourceFile:1)
    at com.aspiro.wamp.tidalconnect.discovery.TcListAvailableDevicesUseCase.getAllAvailableDevices(SourceFile:115)
    at com.aspiro.wamp.tidalconnect.discovery.TcBroadcastProvider.getAllAvailableDevices(SourceFile:3)
```

Reproduced with Tidal Android 2.211.0 (Pixel 7 Pro) and 2.212.0 (Pixel 10) by
others in #6, and on our own Android phone.

## 2. Root cause

`tdcn` advertises `_tidalconnect._tcp` through its own built-in mDNS responder
(the `mdns-sd` 0.11.5 Rust crate) with an **incomplete TXT record**, and an
instance name that doesn't follow the convention real devices use.

What `tdcn` 2.0.5 advertises:

```
ZenStream-<random hex>._tidalconnect._tcp.local.   port 2019
TXT "fn=hifimbr" "id=f765624cd55f11f1b3d12ccf67478fea"
```

What working Tidal Connect devices on the same LAN advertise:

| Device | Instance name | TXT |
|---|---|---|
| hifistreamer (another streamer) | `<mn>-<id>` | `ve=1 fn=… id=… ca=1 mn=…` |
| Samsung HW-LS60D soundbar | `<mn>-<id>` | `ve=1 fn=… id=… ca=5 mn=…` |
| Volumio (from [RPi forum thread][volumio]) | `Volumio-<uuid>` | `mn=Volumio ca=1 id=… fn=Volumio ve=1` |
| Cambridge 851N (same thread) | `851N-<uuid>` | `mn=851N ca=0 id=… fn=851N ve=1` |

Differences:

1. **Missing TXT keys `ve`, `mn`, `ca`.** The Android app evidently dereferences one
   or more of these without a null check. *Which* key(s) are strictly required has
   not been bisected yet (see §7).
2. **Instance name** is `ZenStream-<hex>` where the hex is random per start and does
   not match `id`. Real devices use `<mn>-<id>`. Not proven to matter for the crash,
   but it means every tdcn restart looks like a new device to clients.
3. `fn` is derived at startup from the pretty hostname / hostname, so it only follows
   a hostname change after a restart (normal, just noting).

Field meanings (best understanding, not authoritative):

- `ve` – TXT/protocol version, always `1` in the wild.
- `mn` – model name. For tdcn this should match the certificate model
  (`[device] model` in `config.toml`, default `ZenStream`).
- `ca` – capability flags. Values seen: 0, 1, 5. The Volumio thread ties it to MQA
  support. tdcn has no MQA, so **`ca=0` is the honest value**. Both 0 and 1 have
  been confirmed to stop the crash.
- `id` – 32 hex chars. tdcn derives it from `/etc/uuid` with dashes removed.
- `fn` – friendly name shown in the app.

## 3. What tdcn is (provenance)

`tdcn` is **not** in this repo as source. It is a pre-built, **proprietary**,
stripped arm64 binary (`vendor/bin/tdcn.gz`, copyright "the tdcn authors" per
`debian/copyright`). It appeared in a single upstream commit,
`bde8f384b6` "migrated to a native tdcn wrapper" (Pulpier, 2026-09-09), which
replaced the earlier Docker-based approach (a port of
[TonyTromp/tidal-connect-docker][docker] running iFi's original armhf binary).
No public repo for the tdcn source exists on pulpier's GitHub or in a GitHub search.
It is almost certainly pulpier's own closed code, but that is an inference.

`tdcn-ssl` is listed as MIT but its source isn't published either.

Despite the commit message, tdcn is a **full Rust reimplementation** of the
Tidal Connect device side, not a wrapper around iFi's binary. iFi's
`tidal_connect_application` + `certificate.dat` are only used by `tdcn-ssl` to
recover the device key/certificate at startup.

### Build fingerprint (from the `.comment` section of the vendored binary)

The installed `/usr/bin/tdcn` has `.comment` stripped by `dh_strip`. The vendored
copy in `vendor/bin/tdcn.gz` keeps it. After `strip --remove-section=.comment
--remove-section=.note --strip-unneeded` the vendored binary is byte-identical to
the installed one (sha256 `53b27a2d…226f`), so the repo and the release match.

- `rustc version 1.97.1 (8bab26f4f 2026-07-14)`
- `GCC: (Debian 12.2.0-14+deb12u1) 12.2.0`, i.e. Debian 12 bookworm toolchain
- `clang version 22.1.0-rc2`, probably from Rust's prebuilt std, not a user choice
- Cargo home `/usr/local/cargo`, the layout of the official `rust` Docker image.
  Most likely built in `rust:1.97.1-bookworm` with `cargo build --release`.
- Target `aarch64-unknown-linux-gnu`, PIE, stripped, panic=unwind. Links only
  `libasound`, `libgcc_s`, `libm`, `libc` and needs glibc ≤ 2.34.
- `tdcn-ssl` has the identical toolchain fingerprint.

### Workspace layout (from panic/log paths in the binary)

| Crate | Contents |
|---|---|
| `crates/tdcn` | `main.rs`, `app.rs`, **`discovery.rs`** (the bug), `tls.rs`, `ws.rs` |
| `crates/tdcn-core` | queue/transport: `core`, `load_queue`, `navigate`, `modes`, `advance`, `track`, `transport` |
| `crates/tdcn-proto` | message types (`media.rs`) |
| `crates/tdcn-tidal` | Tidal API: `oauth`, `playback_info`, `queue` |
| `crates/tdcn-audio` | `symphonia` decode → ALSA sink |
| `crates/tdcn-ac` | HiFiBerry audiocontrol bridge |

Notable crates: `mdns-sd 0.11.5`, `rustls` + `aws-lc-rs`, `axum 0.7`, `hyper 1.x`,
`symphonia`, `alsa 0.9`, `clap 4.6`.

### Config keys tdcn accepts

From strings in the binary, matching `debian/tdcn-config.toml`: `model`,
`friendly_name`, `id`, `port`, `binary`, `dat`, `ssl_handler`, `backend`, `device`,
`enabled`, `base_url`, `player_name`, `command_port`, `client_id`. **There is no
key to disable mDNS or to add TXT entries.**

### Consequence

We **cannot fix tdcn itself**: there is no source, and its license is proprietary,
so we can't distribute a patched binary from this fork either. Hand-patching the
compiled `HashMap` construction would be impractical anyway. The real fix has to come
from upstream.

## 4. Fix options

| # | Option | Who | Verdict |
|---|---|---|---|
| 1 | Add `ve=1`, `mn=<model>`, `ca=0` to `txt_properties` in `crates/tdcn/src/discovery.rs`; name the instance `<model>-<id>` | upstream only | **The real fix.** Small change. Request it on #6. |
| 2 | Binary-patch tdcn | us | Rejected: impractical and license-incompatible |
| 3 | Packaging workaround: suppress tdcn's mDNS and publish a correct record via avahi | us, in this fork | **Do this now** as an interim fix (details below) |
| 4 | Switch to TonyTromp's Docker project | — | Rejected (see §6) |

### Option 3 details: suppressing tdcn's own advertisement

`tdcn` runs as a **systemd user service** (`systemctl --user`, uid 1000). That rules
out systemd's `SocketBindDeny=` / `IPAddressDeny=`, which need cgroup BPF and so a
privileged manager.

| Method | Persistent | Needs root at runtime | Scope | Verdict |
|---|---|---|---|---|
| nft rule: drop UDP dport 5353 from uid 1000 (current workaround) | no, lost on reboot unless added to nftables.conf | yes | **all** of uid 1000's mDNS, so it would also kill e.g. librespot's own responder if Spotify Connect is enabled | Replace |
| **`LD_PRELOAD` shim making `bind()` to port 5353 fail with `EADDRINUSE`** | yes, via the unit's `Environment=` | no | tdcn only | **Chosen**, tested OK |

Shim test (2026-11-01, on the Pi): with the shim preloaded, tdcn started normally,
listened on `0.0.0.0:2019` (wss) and `127.0.0.1:8090` (audiocontrol commands),
registered with audiocontrol, logged `device ready`, and opened **no** socket on
5353. It still logs `advertising over mDNS`, which is misleading but harmless. On
shutdown mdns-sd logs two `sending on a closed channel` errors (its daemon thread
had already died), also harmless.

Shim source (tested):

```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <errno.h>
#include <netinet/in.h>
#include <sys/socket.h>
int bind(int fd, const struct sockaddr *a, socklen_t l) {
    static int (*real)(int, const struct sockaddr *, socklen_t);
    if (!real) real = dlsym(RTLD_NEXT, "bind");
    if (a) {
        in_port_t p = 0;
        if (a->sa_family == AF_INET)  p = ((const struct sockaddr_in *)a)->sin_port;
        if (a->sa_family == AF_INET6) p = ((const struct sockaddr_in6 *)a)->sin6_port;
        if (p == htons(5353)) { errno = EADDRINUSE; return -1; }
    }
    return real(fd, a, l);
}
```

Build: `gcc -O2 -shared -fPIC -o block-mdns.so block-mdns.c -ldl`.

### Option 3 details: publishing the correct record

| Method | Notes | Verdict |
|---|---|---|
| Static `/etc/avahi/services/tidalconnect.service` (current workaround) | Needs root to write, so it can't be generated by the user service. Goes stale on hostname/config change. Keeps advertising when tdcn is stopped. | Replace |
| **Companion user unit running `avahi-publish-service`** | Unprivileged (avahi over D-Bus). `BindsTo=tdcn.service`, so the advert exists exactly while tdcn runs. Reads the same config/defaults tdcn uses. | **Chosen** |

The companion helper must reproduce tdcn's defaults exactly:

- `model`: `[device] model`, default `ZenStream`
- `id`: `[device] id`, else `/etc/uuid` lower-cased with dashes removed, else `0`
- `friendly_name`: `[device] friendly_name`, else `PRETTY_HOSTNAME` from
  `/etc/machine-info`, else `/etc/hostname` with the domain trimmed
- `port`: `[device] port`, default `2019`

It then runs:

```
avahi-publish-service "<model>-<id>" _tidalconnect._tcp <port> \
    ve=1 fn=<friendly_name> id=<id> ca=0 mn=<model>
```

Python 3 (`tomllib`, stdlib since 3.11) is the easiest way to parse the TOML.
New package deps would be `avahi-utils` and `python3`.

### Planned fork changes (not yet written)

- `src/block-mdns.c` → built in `debian/rules` → `/usr/lib/hifiberry-tidal-connect/block-mdns.so`
  (package becomes partly compiled. The release workflow already builds on a native
  arm64 runner).
- `debian/tdcn.service`: `Environment=LD_PRELOAD=/usr/lib/hifiberry-tidal-connect/block-mdns.so`
- New `debian/tdcn-mdns.service` (user unit, `BindsTo=`/`After=`/`WantedBy=tdcn.service`)
  plus a helper script, e.g. `/usr/lib/hifiberry-tidal-connect/tdcn-advertise`
- `debian/control`: add `avahi-utils`, `python3` to Depends; `debian/postinst`/`prerm`
  handling for the new user unit, mirroring what's done for `tdcn.service`
- `debian/changelog`: 2.0.6 entry explaining it's interim until tdcn fixes discovery
- README and config comments updated
- Remove everything once upstream ships a fixed tdcn

## 5. Current state of the test device (hifimbr)

Interim manual workaround, applied 2026-10-31 and confirmed working with the
Android app:

- `/etc/avahi/services/tidalconnect.service` publishes
  `ZenStream-f765624cd55f11f1b3d12ccf67478fea`, port 2019,
  TXT `ve=1 fn=hifimbr id=f765624cd55f11f1b3d12ccf67478fea ca=1 mn=ZenStream`
- nft table `inet tdcnblock` drops UDP dport 5353 from uid 1000. **Not persistent.**
- Scripts: `~/tidal-connect-workaround/tidal-fix.sh` and `tidal-undo.sh` on the Pi
  (run with sudo in a real terminal)

When the packaged fix is installed, run `tidal-undo.sh` first.

## 6. Comparison: TonyTromp/tidal-connect-docker

| | pulpier package (tdcn) | TonyTromp Docker |
|---|---|---|
| Engine | Rust reimplementation (tdcn) | iFi's original armhf `tidal_connect_application` |
| Runtime | native arm64 `.deb` | Raspbian Stretch (2017) armhf container, old OpenSSL/curl |
| Target OS | HiFiBerry OS NG (trixie): Web UI + audiocontrol | legacy HiFiBerry OS (Beocreate, AudioControl2) |
| mDNS | own responder, incomplete TXT (this bug) | iFi binary's own, presumably correct |
| Pi 5 | works on the 16K-page kernel | armhf libs are 4K-aligned → needs `kernel=kernel8.img` system-wide (confirmed in pulpier's old `PORTING.md`: "ELF load command alignment not page-aligned") |
| Activity | v2.0.5 2026-09-09, 3★ | last commit 2025-12-21, 208★ |

Both depend on the same ZenStream certificate. TonyTromp's repo already contains
an `IfiAudio_ZenStream.expired`. Not worth switching.

## 7. Open questions / next steps

1. **Bisect the required TXT key(s).** Publish test records with
   `avahi-publish-service` adding `ve`, `mn`, `ca` one at a time and check the
   Android picker. Also test whether the `<mn>-<id>` instance name matters. Makes the
   upstream report sharper.
2. **Comment on #6** confirming the bug on a Pi 5, with the concrete fix (§4, option 1)
   and the evidence table (§2), and ask whether the tdcn source will be published.
3. **Implement option 3 in this fork** (on a branch), build the arm64 `.deb`, install
   on hifimbr, verify with the Android app, verify a reboot, and verify that turning
   the player off in the Web UI removes the advert.
4. Consider: `ca=0` vs `ca=1`. Both stop the crash. 0 is honest (no MQA).
5. Security note: tdcn is an unpublished closed binary holding Tidal OAuth tokens.

[issue6]: https://github.com/pulpier/tidal-connect-hifiberry/issues/6
[volumio]: https://forums.raspberrypi.com/viewtopic.php?t=297771&start=100#p1960617
[docker]: https://github.com/TonyTromp/tidal-connect-docker
