# Device wire protocol

**Status:** v1, 2026-10-02. **This document is the reference.** Other projects that
want devices or agents to connect the same way implement and reference it.

v1 is what the home's servers speak today. The `wire` package in `stackchan-server`
implements it in Go, and Stack-chan's Embody Mode, `tpbot-bridge`, `sbot` and
`stackchan-pet` all use it. Per-device command catalogs live with each device
family (see §8).

## 1. Transport

- **WebSocket**, opened by the device (outbound only; nothing at home needs an open
  port). `ws://` on the LAN today, `wss://` everywhere once nodes have certificates.
- **Path:** `GET /api/workers/connect`.
- **Headers:**
  - `Authorization: Bearer <token>`. v1 uses one shared token per server; v2
    replaces it with a device certificate and a challenge (§9).
  - `X-Yolovm-Worker-Id: <device id>`: 1–64 characters of `[A-Za-z0-9._-]`. The
    header name is kept for compatibility with existing devices; v2 also accepts
    `X-Worker-Id`.

## 2. Frames

- **Text messages** carry JSON frames: `{"kind": "...", "meta": {...}, "body": {...}}`.
  A text message may hold one frame, or a JSON array of frames.
- `meta` may contain `worker_id` (must equal the header), `session_id` and `ts`
  (RFC 3339).
- Frames stay **flat and small**, so a microcontroller can parse them with cJSON or
  ArduinoJson. Servers send one frame per message.
- **Binary messages** carry media and bulk data: the first byte is the type (§6),
  then the payload.
- Unknown kinds and unknown binary types are ignored, so either side can add
  features without breaking the other.

## 3. Session

```
 device                                  server
   │── connect (headers) ───────────────►│
   │── Register ────────────────────────►│   must be the first frame, within 10 s
   │◄─────────────── Accepted / Rejected ─│
   │◄────────────────────────── PairCode ─│   a one-time code to show as a QR
   │◄───────────── Paired (if any viewer) ─│
   │◄──────────────────────── ServerOffer ─│   other servers the device may switch to
   │── RobotTelemetry, RobotEvent ──────►│
   │◄──────────────────────── RobotCommand ─│
   │── binary media (only while asked) ──►│
   │── Heartbeat (every 30 s) ──────────►│
```

- **Liveness:** the server sends a WebSocket ping every 5 s and drops a device after
  60 s without traffic. The device reconnects with backoff from 1 s up to 30 s.
- **Re-registering:** a device whose capabilities change (an extension switched on)
  closes and reconnects with a new `Register`.
- **Local first:** a device keeps a list of endpoints and tries the local ones (a LAN
  alias or IP) before any hub endpoint. It uses a hub endpoint only if its relay
  policy allows it, and returns to the local endpoint as soon as it answers again
  ([architecture §10](architecture.md#10-connection-paths-local-first-w42eu-only-as-a-hub)).
  `ServerOffer` entries are candidates for that list; the owner decides which are
  allowed.

## 4. Frame kinds

| Direction | Kind | Body |
|---|---|---|
| device → server | `Register` | `{"class", "capabilities", "labels"?}` (§5) |
| device → server | `RobotTelemetry` | `{"measurements": {"name": number, …}}` |
| device → server | `RobotEvent` | `{"name", "data"?}`: data values are numbers or strings |
| device → server | `RobotPong` | `{"id", "queue_ms"}`: answer to the `ping` command |
| device → server | `Heartbeat` | `{}` |
| server → device | `Accepted` | `{}` |
| server → device | `Rejected` | `{"reason"}` |
| server → device | `PairCode` | `{"code", "url", "expires_in_s"}`: show `url` as a QR code, `code` as text |
| server → device | `Paired` | `{"viewers", "reconnect"?}`: a browser paired; `reconnect` means "paired before" |
| server → device | `RobotCommand` | `{"command", "args"?}` |
| server → device | `ServerOffer` | `{"servers": [{"name", "url", "token"?}]}` |

The kind names say "Robot" for historical reasons; they apply to every device class.

## 5. Register

```json
{"kind": "Register", "meta": {}, "body": {
  "class": "robot",
  "capabilities": {
    "model": "stackchan-cores3",
    "firmware": "1.5.1",
    "commands": ["ping", "nod", "look", "camera", "car_drive", "..."],
    "measurements": ["battery_pct", "head_yaw_deg", "car_echo_us", "..."]
  },
  "labels": {"with": "stackchan-0a1b2c3d4e50"}
}}
```

- **`class`:** `robot` today; reserved: `sensor`, `phone`, `camera`, `adapter`,
  `ai-agent`.
- **`commands`:** every command the device accepts. Servers and UIs show controls
  only for listed commands.
- **`measurements`:** the telemetry names the device sends.
- **Labels:**
  - `with`: the id of the device this one belongs to (a car linked to a robot).
    Browsers paired with that device may also use this one.
- **Capability sets** are recognized by command names, never by model: a device
  listing `car_drive` has a car, whatever reaches it (a bridge or a robot).

## 6. Binary message types

| Type | Direction | Payload |
|---|---|---|
| `0x01` | device → server | camera frame, JPEG |
| `0x02` | device → server | microphone: uint16 LE sample rate, s16le mono PCM |
| `0x03` | server → device | speaker: same layout as `0x02` |
| `0x04` | device → server | microphone, all channels: uint16 LE rate, uint8 channel count, interleaved s16le |
| `0x05` | device → server | IMU samples: uint16 LE count, then per sample uint32 LE ms and 9 float32 LE (accel m/s², gyro °/s, field µT) |
| `0x06` | device → server | touch frames: uint16 LE count, then per frame uint32 LE ms, uint8 n, n × (uint8 id, uint16 LE x, uint16 LE y) |
| `0x07` | device → server | full-resolution still (JPEG), the answer to `snapshot` |
| `0x08` | device → server | light and proximity samples: uint16 LE count, then per sample uint32 LE ms, uint16 LE proximity, ch0, ch1 |
| `0x10` | server → device | picture to show (JPEG) |
| `0x11` | server → device | file chunk: uint8 name length, name, uint32 LE total size, uint32 LE offset, data |

New types are added here first. `0x80`–`0xFF` are free for experiments.

## 7. Conventions

- **Raw data.** Measurements and events report what the hardware measured or did
  (an echo in µs, a button press), not interpretations.
- **Streams only while wanted.** Media and fast sensors are switched on by the server
  (`camera`, `mic`, `*_stream` commands) while someone watches, and off after. These
  commands come from the server, never from a browser directly.
- **Names:** `snake_case`; units in the name (`_pct`, `_deg`, `_us`, `_mv`).
- **Extensions** use a prefix (`car_*`) and an `*_enable` command that makes the
  device register again with the extension's commands.
- **Safety:** a device that moves stops on its own when commands stop coming (a
  watchdog), and when its server connection drops.

## 8. Command catalogs

The commands a device family offers are documented with that family:

| Family | Catalog |
|---|---|
| Stack-chan | `stackchan-server` readme, "Commands" |
| Car (`car_*`) | `sbot` readme, "Car capability"; firmware side in `tpbot-ble` |

## 9. v2 (planned)

- **Device authentication:** a device certificate signed by the owner key, and a
  challenge signed by the device key, instead of the shared token
  ([architecture §8](architecture.md#8-trust-identity-and-access-control)).
- **Source tags on commands:** every `RobotCommand` carries the `who` (person, app,
  loop or agent) and the `via` (proxy device) it came from, so the device can enforce
  permission entries.
- **Permission entries:** a `Permissions` frame (server → device) delivers the signed
  entries that name the device ([architecture §8.2a](architecture.md#82a-permissions-who--device--capability--time));
  the device checks commands against them, including their time window.
- **Capability descriptors:** per command its arguments, units, scope and safety
  class, so servers, UIs and AI agents can use a new device without code written for
  it.
- **Neutral names:** `DeviceTelemetry`, `DeviceEvent`, `DeviceCommand` accepted next to
  the v1 names.
- **Delegated actions:** `action_request {id, action, params, why, expires}` from a
  requester and `action_result {id, ok, receipt | signature | reason}` from the holder,
  routed by the node to the devices allowed to hold that action
  ([architecture §8.7](architecture.md#87-delegated-actions-secrets-stay-in-their-perimeter)).
  The secret itself never travels in any frame.
- **UI sessions:** a `Session` frame (server → device) with the app and view, the
  person, and the end condition, and `session_start` / `session_end` events back. The
  device's default UI and its return to it stay on the device, so a device without its
  node still shows something safe
  ([architecture §5.1](architecture.md#51-ui-sessions-a-scan-changes-the-devices-ui-then-it-comes-back)).
- **Endpoint announcements and relay policy:** an owner-signed list of the node's
  local and hub endpoints, and the device's relay policy (`never`, `when-away`,
  `always`), delivered to the device and enforced by it.
