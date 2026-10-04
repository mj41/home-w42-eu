# Setting a device up: firmware, connection, apps

**Status:** 2026-10-04, design. Firmware `embody-v0.1.0` is released and approved (§6). A
first version works as one page in the Stackchan manager,
[s-w42-eu-manager](https://github.com/mj41/s-w42-eu-manager) (`/setup`: install, backup,
connect, pair, all in one), tested end to end on a Stackchan. This
document splits it into small parts with clear trust, and keeps the one-page experience
where the user allows it. The apps a device can use are in [App catalog](app-catalog.md).

## 1. Use cases

| Who | Wants | Today (POC) | Target |
|---|---|---|---|
| Anyone with a new robot | plug it in, press one button in Chrome, it works | sm.w42.eu/setup does it all | the same one page, with the firmware step shown and allowed by them |
| Owner with a home server | the same, with their server, Wi-Fi and apps filled in | `localhost:8790/setup` (the home's manager) on its computer | the same; apps from the home's [catalog](app-catalog.md) |
| Owner at another computer | set a robot up for their home server | paste a setup JSON | the same, or scan a code from the home server's page |
| Developer | their own firmware build, then connect it | `container.sh flash`, then `/setup` with "keep firmware" | unchanged |
| Anyone | go back to the robot's original firmware | restore from the backup file on `/setup` | the same, in the flasher |

A robot that already has Embody Mode only needs **connecting**, which stays one click
everywhere. Installing firmware is rarer and riskier, so it gets its own part.

## 2. Threat model

Whatever page holds the robot's USB port can do anything to it: the browser's port chooser is
the only consent, and the chip's ROM flasher cannot be refused by firmware. So the design
limits **which code** asks for the port, and what a robot accepts **without a tap**.

| Party | Can | Must not be able to (principle 4) |
|---|---|---|
| The firmware release pipeline (CI on `embody-v*` tags) | define what "official firmware" is | — it is the trust root for firmware, like the source itself |
| The flasher page (static, published with each release) | install, back up, restore the official firmware | learn tokens, Wi-Fi passwords, accounts |
| A server's connect page (e.g. sm.w42.eu/setup) | give the robot a server, its token, apps, Wi-Fi | change the firmware; redirect a robot without the person at the robot agreeing |
| The relay (raw.sa.w42.eu) at runtime | see metadata, drop messages | read or drive (see [e2ee](e2ee.md)) |
| Someone with the cable | everything | — physical access is the boundary, as with the QR code |

Today's POC still breaks the third row in one way: sm.w42.eu serves the installer **and** the
manifest it checks against, so a compromised sm.w42.eu could flash any robot set up through
it. Making a new server a robot's default needs a Yes on the robot's screen (3.3).

## 3. Parts

```
 firmware repo ──CI──► release: parts + manifest + flasher page   (static, one origin per release)
                                         │  install · backup · restore
                                         ▼
 robot ◄── USB ── browser ── connect page of a server ──► server: tokens, accounts, pairing
   │      (hello · provision · pair, confirmed on the robot)        ▲
   └──────────────── Wi-Fi ──────────────────────────────────────────┘   apps from the catalog
```

### 3.1 Firmware release

One firmware source, build configurations as files (`sdkconfig.defaults` plus
`sdkconfig.defaults.release`), one build script (`release.sh`) for CI and local checks. A
release holds the parts, `manifest.json` with their SHA-256, one merged image and
`SHA256SUMS` (planned: the flasher page). No server, token or Wi-Fi is ever inside: those are
the robot's settings.

Every release is rebuilt by independent builders who sign its hashes in a public repository,
the build is reproducible, and the owner approves it (planned: before it is published): §6.

### 3.2 Flasher

**Planned.** Today s-w42-eu-manager's `/setup` flashes (sm.w42.eu/setup or a manager at
home), with the firmware of the GitHub release (`-firmware-release latest`), every part
checked against the release's manifest.

A static page and a JS module, published **with each release on GitHub Pages** of the
firmware repo (one origin; each release in its own path, kept):

- **Does:** install the release it was published with; back up the robot's current
  firmware first (on by default; sparse read, erased blocks skipped); keep two records on the
  robot: the first backup as its *original* (never replaced) and the latest backup as its
  *previous* firmware (replaced each time); restore either, from its file.
- **Never:** asks for or sees tokens, Wi-Fi, accounts. It talks to no server.
- Firmware and page come from the same origin and version, so there is nothing to fetch
  across origins and nothing a server can swap.
- Standalone it ends with "Connect your robot": a link to sm.w42.eu/setup, or "your own
  server" (its address).

esptool-js quirks today's setup page handles (0.7.0): `hard_reset` does not pulse RTS, so the page resets the
robot itself; `readFlash` leaves the flasher's MD5 packet unread; backups resume on a new
session after a failed block.

### 3.3 USB setup protocol (on the robot)

Lines on the USB serial port, `@stackchan <JSON>`; the reference is the firmware's
`usb_setup.h`.

| Op | Effect | Confirmed on the robot |
|---|---|---|
| `hello` | id, model, firmware, protocol, the original firmware's record | no (read only) |
| `provision` | servers with tokens, the pinned (default) one, autostart, Wi-Fi | **yes**, when it changes the default server: "Connect to *Pet* (192.168.1.10)? ✓ ✗" |
| `pair` | the pairing link the robot shows | no (the same as reading its screen) |
| `restart` | restart into Embody Mode | no |

The tap answers principle 4 for connecting: a page cannot move a robot to another server
unless the person next to the robot agrees. Adding servers that are not the default, and
Wi-Fi, need no tap. Firmware cannot guard the ROM flasher, which is why the flasher is a
separate, static part (3.2).

### 3.4 Connect page (on each server)

Small, the same on every server that sets robots up (today the Stackchan manager, for every
app in its catalog), as a shared JS module plus each server's API:

1. `hello` over USB. No answer: "This robot needs Embody Mode first" (4).
2. The server gives what only it can: a token (an account's robot on a public server, the
   home's robot token on the home server's own computer), and the apps from its
   [catalog](app-catalog.md).
3. `provision` (the robot asks for the tap), `restart`, then `pair`: the page opens the
   pinned app, paired.

Options on the page: autostart (off by default), the app to start with, Wi-Fi.

- **Wi-Fi of this computer** (home server, same computer only): read only when the person
  presses "Use this computer's Wi-Fi", not on page load.
- **Another computer:** the home server's page shows the setup as JSON to copy, or a QR code
  for the other computer's camera. It holds secrets: shown on demand, never logged.

## 4. One page, when the user allows it

The connect page (3.4) can include the flasher (3.2), so a new robot still needs one page:

- The server is configured with **one release** of the flasher: its origin, version and the
  SHA-256 of its module (`-flasher https://…/embody-v0.2.0/flasher.js#sha256=…`). The page
  loads the module with subresource integrity, and the module loads its firmware from its own
  origin. The server serves neither the installer nor the firmware.
- The page **shows** what it would install: "Embody Mode v0.2.0, from github.com/mj41/StackChan
  (release, checksums)", and installs only after the person **allows** it: the button reads
  "Install Embody Mode and connect". A robot that already has Embody Mode skips it.
- Without `-flasher`, or if the person declines, the page links to the standalone flasher
  and continues after it.

This keeps the one-click experience without making the server a firmware source. A
compromised server could still change its own page; the robot's tap (3.3) and, later, signed
manifests (3.1) are what stop it from redirecting or reflashing robots unnoticed.

## 5. What moves where

| Now in s-w42-eu-manager (`-offer`: s-w42-eu-raw) | Goes to |
|---|---|
| esptool-js, install, backup, restore, the firmware routes, `-firmware-dir`, `-firmware-release`, the firmware cache | the flasher, in the firmware repo's release |
| `/setup`: hello, provision, pair, autostart, app choice | stays: the connect page (3.4), as a shared module |
| `-offer` flags | the [app catalog](app-catalog.md): one directory per app |
| a home manager on its own computer: that computer's Wi-Fi (in `/api/me`), `/api/setup/copy` | stays; Wi-Fi only on request |

## 6. Hardening the release

What a person installs must be what the source says, approved by the owner, and checkable
later by anyone, without trusting GitHub Pages, CI or us.

### 6.1 Publishing needs the owner's approval

Planned; today the workflow publishes on the tag and the owner approves afterwards in
`mj41cz-approved` (6.2).

- The release workflow builds on an `embody-v*` tag, then **waits**: the jobs that create the
  GitHub release and deploy GitHub Pages run in a protected environment (`release`) whose
  required reviewer is the owner. Nothing is published until the owner approves it on GitHub,
  after rebuilding it and signing its hashes (6.2).
- Only the owner may push `embody-v*` tags (a tag ruleset), and tags and commits on the
  release branch must be signed.
- The workflow has the least permissions per job, and its actions are pinned by commit SHA.
- GitHub Pages deploys only from this workflow (source: GitHub Actions), never from a branch
  someone can push to.

### 6.2 Signed hashes from independent builders (like Bitcoin Core's guix.sigs)

Copied from Bitcoin Core: every release is rebuilt by independent **builders**, each on their
own machine; each publishes the SHA-256 of every output, signed with their own key, in a public
repository. Nobody's machine has to be trusted alone: a tampered build, or a tampered compiler
on one machine, gives different hashes and stands out.

**`mj41cz-approved`, a public repository on GitLab** (not GitHub, not Akamai; the namespace is
the owner's): append-only, the branch protected against force pushes, commits signed (SSH).
It is the publication: no web page, anyone clones it and checks it with `git` and
`ssh-keygen`. Other projects' releases (server images, other published files) can join later.

Layout and tool: [mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved) (its README).

- **The owner's approval** is their signed `SHA256SUMS`, matching CI's: the owner rebuilt the
  release from the tag (6.4) and got the same bytes. Then (planned, 6.1) the owner approves
  the publishing job on GitHub.
- **Planned: the owner also signs `manifest.json`** with the owner's P-256 key (principle 3;
  `SHA256SUMS` is signed with the SSH Ed25519 key): `manifest.json.sig` would travel with the
  release, so a browser can check it with WebCrypto where the key comes from elsewhere (the
  connect page on sm.w42.eu, §4; later the robot itself).
- **The rule:** a release is approved when the owner's signed `SHA256SUMS` equals CI's and
  every other builder's (today: ci, mj41, cloud). Later: CI's Sigstore attestation too, and
  at least one builder besides mj41's own machines.

### 6.3 Timestamps by third parties

Records by third parties, so neither GitHub, nor one service, nor we can backdate or swap a
release unnoticed:

- **One public timestamp service, for the approval:** an RFC 3161 timestamp of the owner's
  signature (`SHA256SUMS.sig`) from DigiCert's public service (`http://timestamp.digicert.com`:
  free, widely used for code signing, long-lived roots; plain HTTP is fine, the answer is
  signed), as `SHA256SUMS.sig.tsr`, checked with `openssl ts -verify` against the system's CA
  certificates.
- **Sigstore, for the CI build (planned, not in the release workflow yet):** GitHub artifact
  attestations (`actions/attest-build-provenance`) sign each file with a short-lived Sigstore
  certificate bound to the workflow, the repo and the commit, and record it in **Rekor**,
  Sigstore's public transparency log, with its time. Its Rekor URL would go into `ci/source`;
  checked with `gh attestation verify <file> --repo mj41/StackChan`.

### 6.4 Reproducible builds

The strongest check is an independent rebuild that gives the same bytes: then the owner does
not have to trust CI at all. A release build gives the same bytes for all seven published
files (the five parts, `manifest.json`, the merged image) on GitHub Actions, on the owner's
laptop and on a cloud machine.

| | State |
|---|---|
| Toolchain | the ESP-IDF container pinned by digest |
| ESP-IDF managed components | `dependencies.lock` with hashes, checked when fetched |
| Other dependencies (`repos.json`) | pinned by commit (the tag kept for reference); `fetch_repos.py` checks the checkout |
| Paths, dates and times in the binary | none: `CONFIG_APP_REPRODUCIBLE_BUILD=y`, `SOURCE_DATE_EPOCH` from the commit |
| Configuration | from the defaults files only: `release.sh` starts from a fresh `sdkconfig` |
| Version string | `git describe` of the tag |
| Generated assets image | packed in name order (sorted `os.walk`, our patch to xiaozhi) |
| Incremental builds | none for releases: `release.sh` builds clean |

Limits: the same compiler is required (another compiler gives other bytes), and the merged
image depends on that container's esptool too.

**How far down the toolchain is trusted:**

| Level | What is trusted | Result |
|---|---|---|
| 1. Espressif's compiler | their prebuilt toolchain (GCC 14.2.0 for Xtensa, binutils, newlib, picolibc) in the container pinned by digest | GitHub and a laptop: the same bytes |
| 2. Our own compiler | built from Espressif's crosstool-NG sources (`esp-14.2.0_20260121`, every component at its tagged commit), on Ubuntu 22.04 (host GCC 11.4), in Espressif's build path `/builds/idf/crosstool-NG` | **the firmware is the same bytes** as with Espressif's compiler, all seven files |
| 3. Bootstrapped host | the same, but every host tool (compiler, binutils, C library, make, Python, meson, Rust for the wrappers) from Guix 1.5.0, whose packages are built from a small auditable seed; `guix challenge`: all 40 host packages identical on both Guix build farms and here | **the firmware is the same bytes** again |
| Cloud rebuild | level 3 on a short-lived Linode machine with a certified key of its own; the Guix host tools from both build farms, checked by `guix challenge` (not yet built from the seed) | **the same bytes**, with a signed audit log ([Independent rebuild](independent-rebuild.md)) |

Three compilers with different histories give the same firmware: Espressif's compiler adds
nothing that its source does not explain (*diverse double-compiling*, David A. Wheeler).

What the toolchain builds depend on:

- **The build path ends up in the firmware:** newlib's `assert()` messages carry source paths,
  so our toolchains are built in Espressif's path, `/builds/idf/crosstool-NG`.
- **The host compiler that builds the cross compiler** shapes the code of a few target library
  functions (complex maths in `libm`, which the firmware does not link): built with
  Espressif's host GCC 6.3.0, all 195 target library files are byte-identical to theirs.
- **Level 3 is not "from the seed on this machine":** the Guix packages come from Guix's build
  farms (checked with `guix challenge`); `guix build --no-substitutes` would rebuild them, in
  many hours.
- crosstool-NG installs Rust with `curl … | sh` for Espressif's small wrapper programs; the
  Guix build uses Guix's Rust instead (a two-line patch).

The scripts that rebuild the toolchain (levels 2 and 3) are in the firmware repository,
`firmware/toolchain/`. A third verification on a short-lived cloud VM (Linode): Espressif's
toolchain built from source with Guix host tools, a certified key of its own, a signed audit
log: [Independent rebuild](independent-rebuild.md).

The Python tools (esptool, the ESP-IDF build scripts) and CMake shape the output too; they are
source, pinned in the same container, and readable. `fetch_repos.py` stops when a patch
does not apply (an already applied one is fine), so no release is built without it.

### 6.5 Upstream fixes

What the reproducibility work found goes back upstream. Each fix lives first in our fork of the
upstream repository, on the same branch everywhere, **`repro-mj41cz`**; our builds use those
branches, everything is tested end to end (two toolchain builds in different paths, the release on
GitHub and a laptop, the setup page on a robot), and the pull requests go out together with the
blog post about the [independent rebuild](independent-rebuild.md).

| Upstream | Fix on `repro-mj41cz` | Our workaround today |
|---|---|---|
| espressif/crosstool-NG | target libraries built with `-ffile-prefix-map=<build dir>=.`: the toolchain no longer depends on being built in `/builds/idf/crosstool-NG` | build in that path |
| espressif/crosstool-NG | the wrapper programs built with a host `cargo` when given (`CT_SYSTEM_CARGO`) instead of `curl … \| sh` rustup | `system-cargo.patch` |
| espressif/esp-idf | with `CONFIG_APP_REPRODUCIBLE_BUILD`, `SOURCE_DATE_EPOCH` set from the project's git commit | `release.sh` sets it |
| espressif/esptool-js | `hard_reset` pulses RTS (raises it first) | the setup page resets the robot itself |
| espressif/esptool-js | `readFlash` reads and checks the flasher's MD5 packet | the setup page reads per block and compares with the scan's MD5 |
| 78/xiaozhi-esp32 | `build_default_assets.py` walks directories in name order | our patch |
| Forairaaaaa/mooncake | the build banner honours `SOURCE_DATE_EPOCH` (or goes) | `SOURCE_DATE_EPOCH` |
| espressif/gcc | an issue, with a minimal reproducer: the cross compiler's code for some functions depends on the host compiler that built it (GCC 6.3 versus 11 and 14) | — |

### 6.6 On the robot, later

- **Signed manifests** (principle 3): the flasher and the robot accept only firmware whose
  manifest the release key, later the owner key, signed.
- **Secure boot** (ESP32-S3, opt-in, irreversible): only firmware signed with the owner's key
  boots.
- **Flash encryption** (opt-in, irreversible): a robot's tokens and Wi-Fi passwords cannot be
  read from its flash. It also ends plain backups and USB flashing of unsigned images, so it
  is for owners who give up tinkering.

## 7. Rollout

1. **Done:** firmware: `provision` asks for a tap when it changes the default server.
2. **Open:** flasher: move install/backup/restore into the firmware repo, publish it with the
   release (CI), standalone page first.
3. **Open:** s-w42-eu-manager: drop the firmware parts, link to the flasher; Wi-Fi on request.
4. **Open:** `-flasher` with a pinned release: one page again, with the person's consent.
5. App catalog: **done** as a design ([its own doc](app-catalog.md)); **open:** the catalog
   itself, then the connect page as a shared module for the pet and sbot.
6. Release hardening (§6): **done:** the GitHub-versus-local rebuild check, the
   `mj41cz-approved` repository on GitLab with its tool, the timestamp, our own compiler
   (levels 2 and 3) and the cloud rebuild; **open:** the `release` environment with the owner
   as reviewer, the Sigstore attestation, the signed manifest.
7. **Open:** signed manifests (principle 3).

## 8. Decisions

- **The tap for `provision`** is asked only when the default server changes. Servers that
  are not the default do nothing until someone chooses them, and the robot shows its list.
- **Backups (done):** the robot records its original firmware (once, `orig_fw`) and the
  firmware before the latest setup with a backup (replaced each time, `prev_fw`); the setup
  page restores either. Older backups are files, restored with esptool from a terminal.
- **Timestamp service:** DigiCert's public RFC 3161 service, for the owner's signature
  (§6.3).
- **Approved artifacts, like Bitcoin Core's guix.sigs:** a public repository on GitLab,
  `mj41cz-approved` (not GitHub, not Akamai): signed `SHA256SUMS` per builder per release, CI's
  hashes with their source, and the public Go tool. No separate web page. The owner also
  signs `manifest.json` for browsers (§6.2).
- **Toolchain:** levels 1–3 give identical bytes; releases are built with level 1 and checked
  against levels 2–3 and the cloud rebuild (§6.4).
- **Flasher:** GitHub Pages of the firmware repository (§3.2), planned; today
  s-w42-eu-raw's `/setup`.
