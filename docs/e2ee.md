# End-to-end encryption between a device and its browsers

**Status:** 2026-10-03. Done: steps 1 and 2 of the rollout (§9): the Go reference
implementation (stackchan-server's `e2e` package, test vectors), the relay, and the browser
side (the dashboard's `e2e.js`, the vectors pass in Chrome), tested end to end with
`fake-robot -e2e`. Next: the firmware.
Part of the [wire protocol](wire-protocol.md) (planned for v2, usable from v1 as an extension).

A relay such as `chan.w42.eu` connects robots and browsers that cannot reach each other
directly. Today it sees everything: camera frames, microphone audio, telemetry, commands.
With end-to-end encryption it carries only ciphertext between a robot and the browsers its
owner enrolled, so a relay that is curious, compromised or compelled cannot watch, listen
or drive.

## 1. Threat model

| The relay (server) can | It cannot |
|---|---|
| see that a robot is online, when, from where, and how much it sends | read camera frames, audio, telemetry, events or commands |
| drop, delay or reorder messages | inject a command the robot accepts, or replay one |
| see the robot's capability list (Register) and pairing codes | enroll its own key with the robot, or pose as the robot to a browser |
| switch streaming on and off (`camera`, `mic`) | see what is streamed; the robot's LIVE badge still shows it |

Out of scope: a compromised robot or browser device; someone who photographs the robot's
QR code (they can enroll, as they can pair today: physical access is the boundary).

## 2. Keys

| Key | Who has it | Lifetime |
|---|---|---|
| **Robot key** `R` (X25519) | the robot (NVS) | until reset; its public half travels in the QR code |
| **Pairing secret** `P` (16 random bytes) | the robot, and the QR code's URL fragment | one QR code: replaced with every pairing code |
| **Browser key** `B` (X25519) | one browser (IndexedDB, private part non-extractable) | until the browser forgets it |
| **Pairwise key** `K_B` | the robot and browser B | derived: HKDF(X25519(R, B)) |
| **Group key** `G` (AES-256) | the robot and every enrolled browser | an *epoch*; new on boot and whenever a browser is removed |

The relay never sees `P` (URL fragments are not sent to servers), the private halves, `K_B`
or `G`.

Primitives: X25519, HKDF-SHA256, HMAC-SHA256, AES-256-GCM with 96-bit nonces. All are in
browsers' WebCrypto and in mbedtls (ESP32-S3 has hardware AES).

## 3. Enrollment: the QR code is the trusted channel

The robot's pairing QR code gets a fragment:

```
https://chan.w42.eu/pair?code=KA553ZQC#e2e=1.<R_pub b64url>.<P b64url>
```

1. The browser pairs as today (`/pair?code=…`, the session cookie). The fragment survives the
   redirect to the dashboard, which reads it and removes it from the address bar. The
   **robot** appends the fragment to the URL it draws as its QR code: the server sends the
   URL without it and never learns it. Typing the 8-character code instead of scanning pairs
   without encryption (there is no fragment), so an encrypted robot asks for the QR code.
2. The browser creates (or reuses) its key `B` and sends, through the relay,
   `E2EEnroll {"b": B_pub, "mac": HMAC-SHA256(P, "w42-e2e-enroll|" + R_pub + "|" + B_pub)}`.
3. The robot checks the MAC against the current `P` (and the one before, for a QR code that
   changed while scanning), stores `B_pub` in its enrolled list, and makes a new `P` for its
   next QR code: each `P` enrolls one browser.
4. Both derive `K_B = HKDF-SHA256(ikm = X25519(R, B), salt = R_pub || B_pub,
   info = "w42-e2e pairwise")`.
5. The robot sends the group key: `E2EGroupKey {"b": B_id, "epoch": e, "n": nonce,
   "c": AES-GCM(K_B, G, aad = "w42-e2e group|" + robot id + "|" + e)}`.

Why the relay cannot cheat: it does not know `P`, so it cannot enroll a key of its own; the
browser takes `R_pub` from the QR code, not from the relay, so the relay cannot pose as the
robot. `B_id` is the first 8 bytes of SHA-256(`B_pub`), hex.

A browser that comes back sends `E2EHello {"b": B_id}`; the robot answers with the current
group key under `K_B` if `B` is still enrolled.

## 4. Robot to browsers: one ciphertext for all

Everything the robot sends to browsers goes out once, under the group key:

- **Binary** (camera, microphone, IMU, touch, snapshots): message type `0x30`,
  `0x30 | epoch u32 BE | nonce (12) | AES-GCM(G, inner type byte + payload)`, with
  aad = robot id. The inner type is the plaintext type (`0x01` video, `0x02` audio …).
- **Telemetry and events:** frame kind `E2EData {"epoch", "n", "c"}`, where `c` decrypts to
  the frame the robot would have sent in plaintext (`Telemetry`, `Event`, …).
- **Nonces:** 4 random bytes chosen at boot plus a 64-bit counter: never repeated under one
  key, and a new epoch starts on boot anyway.

## 5. Browser to robot: per browser

Commands are sealed under the browser's pairwise key, so the robot knows who sent them:

`E2ECommand {"b": B_id, "n": nonce, "c": AES-GCM(K_B, {"command", "args", "seq"},
aad = robot id)}`. The robot accepts a `seq` only if it is higher than the last one from
that browser (no replays). Speaker audio uses binary type `0x31`:
`0x31 | B_id (8) | nonce (12) | AES-GCM(K_B, payload)`.

## 6. The relay's part

- It forwards `E2E*` frames and binary `0x30`/`0x31` between the robot and the browsers
  paired with it, without understanding them, and keeps doing what needs no plaintext:
  pairing, sessions, who may receive what, rate limits, `camera`/`mic` on while watched.
- What it can no longer offer for an encrypted robot: telemetry in its state file, the
  event replay for late browsers, snapshots kept on the server, command history. The
  dashboard keeps that state in the browser instead.

## 7. Per server, by the robot

Encryption is a setting of each entry in the robot's server list, because it only fits a
**relay** whose app runs in the browser (the dashboard). App servers that are the app
(the pet, sbot) need the data and run on the owner's own network; they stay plaintext.
`chan.w42.eu` is a relay: encrypted. With encryption on, the robot sends no plaintext media
or telemetry to that server and accepts commands only as `E2ECommand`, except the relay's
`camera`/`mic` stream switches.

Also refused from the relay while encrypted: plaintext binary messages (pictures, file
chunks, speaker audio), since the relay could inject them. Pictures and files go sealed in
a later step; until then they are not available for an encrypted robot.

## 8. Managing enrolled browsers

On the robot: the QR screen shows how many browsers are enrolled and offers **Forget all**.
From an enrolled browser: `forget {"b": B_id}` (sealed like any command). Forgetting a
browser starts a new epoch, so it cannot read anything sent afterwards.

## 9. Rollout

1. This design and a Go reference implementation (key derivation, sealing, enrollment, test
   vectors) used by stackchan-server's relay and `fake-robot`, tested end to end without
   hardware.
2. Browser side in the dashboard (WebCrypto), tested against `fake-robot`.
3. Firmware (mbedtls), behind a per-server setting, off until tested on the robot.
4. On for chan.w42.eu.
