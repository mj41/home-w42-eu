# Embody Mode: trust and connection design

Status: **draft for review**. What exists is under "Today"; the rest is the plan.

## Today

- **Connection:** Embody Mode connects to [s-w42-eu-raw](https://github.com/mj41/s-w42-eu-raw) (or another app server) with a bearer token. Browsers pair by scanning the robot's QR code.
- **Tokens:** the release firmware ([embody-v0.1.0](https://github.com/mj41/StackChan/releases/tag/embody-v0.1.0)) has no server or token: they are written into NVS over USB at setup. On chan.w42.eu each robot gets its own token (an account adds the robot; the server keeps only the token's hash). A home server's own robots still share one token.
- **People:** chan.w42.eu signs people in through Dex at auth.w42.eu (GitHub, Google); a robot belongs to the account that added it and is private to it by default. Tiers set rate limits, not access.
- **End-to-end encryption** per server: with it on, the relay carries only ciphertext ([e2ee](https://github.com/mj41/home-w42-eu/blob/main/docs/e2ee.md)).
- **Releases:** reproducible (CI, the owner's laptop and a cloud rebuild give the same bytes), approved by the owner's signed hashes ([mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved)).
- **Firmware update path:** the xiaozhi OTA server check is disabled, and firmware is never installed from a server. See "Firmware work" below.
- **Problem:** anyone who dumps a robot's flash gets its token and can act as that robot. There are no scopes yet: every paired browser can use every capability, including camera and microphone (the robot shows a LIVE badge while they stream).

## Goals

1. **Each owner is their own root of trust.** An owner holds one key. It signs everything about their robots: firmware, config, which apps may do what.
2. **The w42 services are not trusted for authenticity.** A compromised `chan.w42.eu` must not be able to redirect a robot to a server the owner didn't choose, or impersonate a robot.
3. **One universal firmware image.** Anyone can rebuild it from source, check the hash, and sign it with their own key.
4. **Private use stays local.** The owner's own apps run on the owner's server on the LAN, and the cloud never sees that traffic.
5. **Physical security is the baseline; secure boot comes later.** The tool can re-verify a robot's flash at any time, which gives detection now and prevention later.

## Roles

```
                 owner laptop: owner key + chanctl
                   | signs: firmware manifest, robot cert, config, grants, server cert
                   v
  robot ------ (1) rendezvous -----> chan.w42.eu      only rendezvous; holds owners' public keys
    |  \
    |   \----- (2a) our public apps -> appchan.w42.eu   our apps, only as granted; browsers from anywhere
    |
    \--------- (2b) private apps ----> owner's embody server on a private IP (LAN)
                                          browsers on the LAN
```

| Component | Role | Holds |
|---|---|---|
| **Owner key** | Root of trust. ECDSA P-256, on the laptop, on a YubiKey (PIV), or later as a passkey. | Private key: owner only |
| **chanctl** (Go, planned) | Owner tool that gets, signs, writes and verifies firmware, and signs config and grants. | Uses the owner key |
| **Robot** | Enforces everything: verifies signatures with the owner public key it carries, and enforces scopes. | Owner public key, its own device key, its owner-signed robot certificate, signed config and grants |
| **chan.w42.eu** | Rendezvous. It authenticates robots against the owner keys it knows and hands out owner-signed endpoint info. | Owners' **public** keys (config file), online status |
| **appchan.w42.eu** | Hosts **our** apps. A robot connects only to the apps its owner granted. | App code; no owner secrets |
| **Owner's embody server** | Private apps on the owner's LAN. It is `s-w42-eu-raw` in the "embody" role. | Its own key and an owner-signed server certificate |

## Keys, certificates and signed documents

- **Owner key: ECDSA P-256.** The robot's mbedTLS verifies it, YubiKey PIV and passkeys support it, and Go supports it natively. Ed25519 is not an option because mbedTLS lacks it.
- **Certificates: X.509, with the owner key acting as a small CA.**
  - **Robot certificate:** binds the robot ID to the robot's device public key.
  - **Server certificate:** binds the owner's embody server to its key.
- **Robot device key:**
  - **v1:** a software key generated on the robot and kept in NVS, to get the flow working.
  - **Target:** the ESP32-S3 **DS peripheral**. It is RSA-3072 only, and the private key never leaves the chip.
- **Signed documents** are JSON (the robot parses them with ArduinoJson), each with a detached ECDSA P-256 signature over the **exact file bytes**, so no canonicalization is needed. Owners may write YAML; chanctl converts it to JSON before signing. Every document has a `serial`. The robot rejects anything with a serial at or below the last one it accepted, which prevents rollback.

| Document | Signed by | Content |
|---|---|---|
| Firmware manifest | owner | robot ID, app image sha256, assets sha256, provisioning payload sha256, serial |
| Robot config | owner | rendezvous URL (`https://chan.w42.eu`), private endpoint policy, serial |
| Grants | owner | robots, apps (on appchan) with scopes and app version pins, expiry, serial |
| Endpoint announcement | owner's embody server (its cert chains to owner) | current private URL(s), e.g. `wss://192.168.1.10:8765`, timestamp |

## Scopes

| Scope | Allows |
|---|---|
| `passive.basic` | robot id, model, battery, online status |
| `passive.all` | plus all sensors: IMU, touch, head angles, Wi-Fi, … |
| `active.basic` | safe gestures: nod, shake, expressions |
| `active.all` | full motion, avatar, speech, settings |
| `media.video` | camera |
| `media.audio-in` | microphone |
| `media.audio-out` | speaker |

- `.all` implies `.basic`.
- `media.*` is never implied and must be granted explicitly.

How today's commands and data map onto scopes. This is the table the robot will enforce:

| Scope | Commands | Data to the browser |
|---|---|---|
| `passive.basic` | `ping` | id, model, battery, charging, online |
| `passive.all` | | head angles, Wi-Fi, memory, uptime, brightness, volume; events (shake, head touch, screen taps) |
| `active.basic` | `nod`, `shake`, `emotion`, `sticker` | |
| `active.all` | `look`, `home`, `say`, `leds`, `brightness`, `volume`, `face`, pictures (`image`) | |
| `media.video` | `camera` | camera frames |
| `media.audio-in` | `mic` | microphone audio |
| `media.audio-out` | not built yet (voice from phone to robot) | |
- Private apps on the owner's own server get everything by default, because the owner is the operator. Owners can still restrict them in the config.

Example grants (before conversion to JSON and signing):
```yaml
owner: mj41-2026
serial: 7
expires: 2027-03-29
robots: [stackchan-0a1b2c3d4e50]
apps:
  - app: https://appchan.w42.eu/dashboard
    version_sha256: 3f9a…
    scopes: [passive.basic, active.basic]
  - app: https://appchan.w42.eu/telepresence
    version_sha256: 81c2…
    scopes: [passive.all, active.all, media.video, media.audio-in]
```

## Flows

### A. Owner setup (once)
1. `chanctl owner init` creates the owner key and prints the public key and its fingerprint.
2. The owner registers the public key with `chan.w42.eu`. In v1 that is an entry in the owners config file (deployed with the server); later it can be self-service, where the owner proves possession of the key.
3. For a private server: `chanctl server cert` issues the owner-signed server certificate.

### B. Provisioning a robot (chanctl full cycle)
1. **get:** download a release, or build it from source. With reproducible builds the hashes match, so anyone can check that a release really is that source.
2. **sign:** create the provisioning payload: owner public key, the robot certificate (whose device key comes from step 4 on first provisioning), and the signed config and grants. Then sign the firmware manifest.
3. **write:** flash the app, assets and **provisioning partition**. The app image is identical for everyone; only the provisioning partition is per owner and per robot.
4. **device key:**
   - **v1:** the robot generates its key on first boot and reports its public key. chanctl signs the robot certificate and writes it.
   - **DS peripheral:** chanctl burns the key into eFuses during provisioning.
5. **verify:** read the whole flash back through the chip's **ROM download mode** and compare it with the signed manifest.
   - **Why ROM:** running firmware can't be trusted to report its own hash, but the ROM loader is in silicon.
   - **Limit:** it's a snapshot. Re-run `chanctl verify` any time as an audit.

### C. Robot boot and rendezvous
1. The robot checks the provisioning partition: config and grants signatures against the embedded owner key, and serials.
2. It connects to `chan.w42.eu` and checks its TLS certificate with the public CA bundle. That only proves it reached the real host; it does not establish trust.
3. The server sends a random challenge. The robot answers with its robot certificate and a signature over (challenge, `chan.w42.eu`, robot ID) made with its device key.
4. `chan.w42.eu` checks that the certificate chains to a registered owner key, then returns:
   - the latest endpoint announcement from that owner's embody server, if any;
   - where on appchan the granted apps are.
5. The robot verifies the endpoint announcement against its owner key, so `chan.w42.eu` cannot substitute its own endpoint.

### D. Private apps (redirect to a private IP)
1. The robot connects directly to the owner's embody server at the announced LAN URL.
2. Both sides authenticate with owner-signed certificates. On the LAN there is no gateway in between, so this can be mutual TLS with the owner CA. The robot uses the DS key for its client certificate once that exists.
3. Browsers on the LAN open the embody server and pair by QR as they do today. `chan.w42.eu` never sees this traffic.

### E. Our public apps (appchan.w42.eu)
1. The robot connects to appchan and asks for the apps listed in its grants, presenting its robot certificate the same way as in C.3.
2. Browsers use the app from anywhere over HTTPS and pair with the robot by QR.
3. Every frame appchan sends carries the app it came from. **The robot drops any frame outside that app's granted scopes.** appchan also filters, as defence in depth.

### E2. Browser users on appchan (login)
- **Authentication:** an external identity provider through Dex at auth.w42.eu (GitHub and Google; built for chan.w42.eu). No passwords are kept.
- **Authorization stays with the owner.** The signed grants list which users may use which app, and the robot enforces it.
  - Users are keyed by the provider's **stable ID**, not by username or email, since both can change: `github:<numeric user id>`, `google:<sub>`.
  - Effective scopes = the app's granted scopes **∩** the user's scopes.
  - appchan tags every frame with (app, user), and the robot checks both.
- **Guests without login:** the robot's QR code still works as proof of physical presence. The grants can give QR-paired guests a small scope set for a limited time.
- **Private apps on the LAN** keep QR pairing only: being on the LAN plus seeing the robot is the proof.
- **appchan secrets:** the OAuth client secret lives in the hosting's secret store, never in git, like the robot token today.

```yaml
users:
  - id: github:12345678        # numeric GitHub user id, never the login name
    apps:
      https://appchan.w42.eu/dashboard: [passive.basic, active.basic]
  - id: google:109876543210
    apps:
      https://appchan.w42.eu/telepresence: [passive.all, media.video]
guests:                        # QR on the robot, no login
  scopes: [passive.basic]
  ttl: 1h
```

- **What trusting appchan means here:** a compromised appchan could claim to be any user listed in the grants. It still can't exceed the scopes granted to our apps. This is the same trust boundary as app labeling in E.3.
- **Later:** passkeys (WebAuthn) as a login that needs no third party. The owner lists passkey public keys in the grants, and the same mechanism can sign grants from a phone.

### F. Updating grants and revocation
- To change or revoke, sign new grants or config with a higher serial. They are delivered through `chan.w42.eu` or with chanctl, and the robot applies them immediately.
- **Revoking a lost robot:** remove it from the grants and the owner's server config. With the software device key, also treat the robot's key as compromised.

## If a part is compromised

| Compromised | Can | Cannot |
|---|---|---|
| chan.w42.eu | see who is online and their IPs; deny service | redirect robots (announcements are owner-signed); impersonate robots; see private traffic |
| appchan.w42.eu | anything within the scopes owners granted to our apps | exceed those grants; reach private apps |
| owner's embody server | control that owner's robots (it is the owner's) | affect other owners |
| stolen robot, no secure boot | read the Wi-Fi password; reflash it; act as that robot until revoked; **clone it** if it uses the software key | act as other robots; forge config or grants |
| stolen robot with DS key | act as that robot while holding it, until revoked | clone it; extract the key |
| owner key leaked | everything for that owner's robots | other owners |

## Firmware work this needs

1. **Close the update hole.** Done, via `patches/xiaozhi-esp32.patch`:
   - **Check disabled:** AI.AGENT no longer contacts `https://api.tenclass.net/xiaozhi/ota/` (Kconfig `STACKCHAN_XIAOZHI_OTA_CHECK`, default off). It uses the protocol settings stored by its last check.
   - **Stored asset URLs ignored:** asset download URLs left by earlier checks are discarded.
   - **No auto-install:** even with the check on, firmware the server offers is never installed.
   - **MCP tool removed:** `self.upgrade_firmware`, which installed firmware from any URL, is gone.
   - **Future update path:** only images whose hash is in an owner-signed manifest.
2. **Reproducible builds.** Done: `CONFIG_APP_REPRODUCIBLE_BUILD=y` in release builds, ESP-IDF 5.5.4 pinned in a container, dependencies pinned by commit; three independent builds give the same bytes ([device setup](https://github.com/mj41/home-w42-eu/blob/main/docs/device-setup.md) §6).
3. **Provisioning partition:** add it to `partitions.csv`, with a reader and a signature verifier (mbedTLS ECDSA P-256).
4. **Device key and challenge-response** in `embody_client`. Replace the shared token.
5. **Owner-CA TLS for the LAN.** Today xiaozhi's `EspSsl` only uses the public CA bundle. Add a custom CA option and a DS client certificate.
6. **Later: Secure Boot v2 and flash encryption.**
   - On the ESP32-S3, Secure Boot v2 is **RSA-3072 only**, so it needs a separate boot-signing key, itself signed by the owner key.
   - Flash encryption also protects the Wi-Fi password, which is plain text in NVS today (`esp-wifi-connect` `ssid_manager`, NVS encryption off).
   - Enabling "secure download mode" would block the ROM read-back in B.5. That check would then have to move to a signed report from a secure-booted bootloader.

## Stages

Each stage leaves a working system.

1. **Signing primitives:** chanctl owner key, sign and verify of documents; C verification on the robot.
2. **Firmware hygiene:** OTA gate, reproducible build, provisioning partition with owner key and config.
3. **Robot identity:** software device key plus owner-signed robot certificate, and challenge-response on `chan.w42.eu`. **Drop the shared token.**
4. **Rendezvous and private apps:** `s-w42-eu-raw` gets a rendezvous role (chan.w42.eu) and an embody role (LAN) with owner-signed endpoint announcements, and the robot follows the redirect.
5. **appchan.w42.eu:** the current dashboard moves there as the first app, and the robot enforces scopes from grants.
6. **chanctl flash and verify** via ROM read-back.
7. **DS peripheral device key**, then Secure Boot v2 and flash encryption (on a spare board first).

## Open questions

- **Signature container:** raw bytes plus a `.sig` file (proposed), or JWS with ES256?
- **Owner registration on chan.w42.eu:** a config file deployed via GitOps (v1), then self-service.
- **Browser to LAN embody server:** plain `http` on the LAN, or a real hostname (e.g. `home.w42.eu` pointing at the LAN IP) with a Let's Encrypt DNS-01 certificate.
- **appchan app pinning:** `version_sha256` in grants pins what appchan must serve, but the robot can't see what the browser actually ran. Is appchan's word enough, or should browsers check it (SRI)?
- **Owner key as a passkey (WebAuthn, P-256)**, so approvals can be signed on a phone.
- **Browser login:** GitHub and Google work; passkeys next? Should guests via QR be allowed at all on appchan, or only on the LAN?
- **Rate limiting** on challenge attempts (wrong pairing codes and failed robot logins are already limited per address).
- **Several owners per robot, and transferring ownership:** probably "re-provision with the new owner key".
