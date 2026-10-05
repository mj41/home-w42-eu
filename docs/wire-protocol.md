# Device wire protocol

**Status:** version 1 ("v1"), in use since 2026-10-02 by every server and device below.
**This document is the reference.** Other projects that want devices or agents to connect
the same way implement and reference it. **Early stage:** v1 changes without backward
compatibility (principle 27): devices and servers are updated together, and every change is
listed with its date in §10. §9 collects the bigger changes already planned.

v1 is what the home's servers speak today. The `wire` package in [s-w42-eu-raw](https://github.com/mj41/s-w42-eu-raw)
implements it in Go, and Stackchan's [Embody Mode](https://github.com/mj41/StackChan/tree/embody-mj41), `tpbot-bridge`
([tpbot-ble](https://github.com/mj41/tpbot-ble)), [s-w42-eu-sbot](https://github.com/mj41/s-w42-eu-sbot) and [s-w42-eu-pet](https://github.com/mj41/s-w42-eu-pet) all use it. Per-device command catalogs live with each device
family (see §8).

## 1. Transport

- **WebSocket**, opened by the device (outbound only; nothing at home needs an open
  port). `ws://` on the LAN today, `wss://` everywhere once nodes have certificates.
- **Path:** `GET /api/devices/connect`.
- **Headers:**
  - `Authorization: Bearer <token>`: a token shared by a server's own devices, or a token
    of the device's own (an invite, or a device added by an account on a server with
    sign-in), which is good only for that device id. v2 replaces tokens with a device
    certificate and a challenge (§9).
  - `X-Device-Id: <device id>`: 1–64 characters of `[A-Za-z0-9._-]`.

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
   │◄──────────────────────── ManagedApps ─│   a manager's signed app list, if changed
   │── AppsVersion ─────────────────────►│   after applying it
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
- **Apps come from managers:** a device's list of servers (its apps) is set by its
  managers (s-w42-eu-manager: a home's own, or sm.w42.eu), over USB at setup and later by
  `ManagedApps`, which the server relays from the manager. A server never changes that list;
  it may suggest switching to another of the device's apps (the `server_switch` command), and
  the device asks on its screen.

## 4. Frame kinds

| Direction | Kind | Body |
|---|---|---|
| device → server | `Register` | `{"class", "capabilities", "labels"?}` (§5) |
| device → server | `RobotTelemetry` | `{"measurements": {"name": number, …}}` |
| device → server | `RobotEvent` | `{"name", "data"?}`: data values are numbers or strings |
| device → server | `RobotPong` | `{"id", "queue_ms"}`: answer to the `ping` command |
| device → server | `Heartbeat` | `{}` |
| device → server | `AppsVersion` | `{"versions": "<manager id>:<version>,…"}`: the app lists it has now, after a `ManagedApps`; never sealed |
| server → device | `Accepted` | `{}` |
| server → device | `Rejected` | `{"reason"}` |
| server → device | `PairCode` | `{"code", "url", "expires_in_s"}`: show `url` as a QR code, `code` as text |
| server → device | `Paired` | `{"viewers", "reconnect"?}`: a browser paired; `reconnect` means "paired before" |
| server → device | `RobotCommand` | `{"command", "args"?}` |
| server → device | `ManagedApps` | `{"payload", "sig"}`: an app list signed by one of the device's managers, relayed unread; the device checks the signature with the manager's key from its setup |
| both | `E2EEnroll`, `E2EHello`, `E2ECommand`, `E2EGroupKey`, `E2EData` | end-to-end encryption between a device and its browsers, passed on unread by the server ([e2ee](e2ee.md)); a device without it ignores them |

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
| `0x30` | device → browsers | sealed under the group key, passed on unread by the server: see [e2ee](e2ee.md) §4 |
| `0x31` | browser → device | sealed under the browser's pairwise key, passed on unread by the server: see [e2ee](e2ee.md) §5 |

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
| Stackchan | s-w42-eu-raw readme, [Commands](https://github.com/mj41/s-w42-eu-raw#commands) |
| Car (`car_*`) | sbot readme, [Car capability](https://github.com/mj41/s-w42-eu-sbot#car-capability); firmware side in [tpbot-ble](https://github.com/mj41/tpbot-ble) |

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
- **Neutral names:** `DeviceTelemetry`, `DeviceEvent`, `DeviceCommand` replace the `Robot…`
  names (renamed, not kept alongside).
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

## 10. Changes to v1

| Date | Change |
|---|---|
| 2026-10-02 | v1: this document. |
| 2026-10-02 | The connect path `/api/devices/connect` and the header `X-Device-Id`. |
| 2026-10-03 | Tokens of a device's own (invites; devices added by accounts), next to a server's shared token (§1). |
| 2026-10-03 | End-to-end encryption frames (§4) and binary types `0x30`, `0x31` (§6), [e2ee](e2ee.md): optional on both sides. |
| 2026-10-05 | `ServerOffer` removed: a device's apps come only from its managers; `ManagedApps` and `AppsVersion` (§3, §4). |
