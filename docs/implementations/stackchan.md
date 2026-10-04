# Stackchan: the first device family

**Status:** 2026-10-04. Works on the LAN and through chan.w42.eu; firmware `embody-v0.1.0`
released and approved ([device setup](../device-setup.md) §6).

M5Stack's Stackchan robots (a CoreS3 with an ESP32-S3, in a body with two servos) is the
first device built the home-w42-eu way: **one universal firmware that is a light
client, and several apps on the server side that give it different jobs.**

Setting up a robot: plug it in and press one button on
[chan.w42.eu/setup](https://chan.w42.eu/setup) (Chrome, USB); with your own server or your
own build:
[SETUP.md](https://github.com/mj41/StackChan/blob/embody-mj41/firmware/main/apps/app_embody_mode/SETUP.md)
in the firmware fork.

## The device

| Part | What the platform gets |
|---|---|
| Screen (320×240, touch) | the face, pictures, sprites, QR codes; raw touch points |
| Head servos (yaw, pitch) | `look`, gestures, hold, continuous yaw; raw positions, load, temperature |
| Camera (GC0308) | live JPEG stream while watched (320×240 or 640×480), 640×480 snapshots |
| Microphone, speaker | raw PCM both ways while in use |
| 12 RGB LEDs, power LED | colours, per-LED pixels, effects |
| IMU, magnetometer | raw accel / gyro / field at 100 Hz while watched |
| Head touch, light, proximity | raw zones, lux, proximity counts |
| NFC reader | tag UID, type and NDEF content |
| IR transmitter and receiver | raw timings, NEC codes |
| Power (AXP2101, INA226) | voltages, currents, plug and button events |
| BLE (NimBLE central) | hosts other devices: the TPBot car today |

Details and coverage: [HARDWARE.md](https://github.com/mj41/StackChan/blob/embody-mj41/firmware/main/apps/app_embody_mode/HARDWARE.md) in the firmware fork; how it works, by example: [Stackchan concept](stackchan-concept.md).

## The firmware: Embody Mode as a light client

The [fork](https://github.com/mj41/StackChan/tree/embody-mj41) of [M5Stack's firmware](https://github.com/m5stack/StackChan) (branch `embody-mj41`) adds **Embody
Mode**, the first launcher app. It:

- connects out to a server over WebSocket and registers its capabilities (the
  [wire protocol](../wire-protocol.md));
- sends raw telemetry and hardware events, and streams media only while someone
  watches or listens (with a red LIVE badge);
- runs commands, and keeps no app logic of its own;
- keeps a **list of servers** and switches with Next / Pin / Connect on its QR
  screen, or by `server_switch`;
- is **set up over USB** (`@stackchan` lines: `hello`, `provision`, `pair`, `restart`); a new
  default server needs a Yes on its screen, and it records its original and previous
  firmware for a restore ([device setup](../device-setup.md) §3.3);
- has **end-to-end encryption** per server: it seals media, telemetry and events for the
  browsers enrolled through its QR code, and from that server takes commands only sealed by
  them (the relay may still switch streams on and off) ([e2ee](../e2ee.md));
- has **optional extensions**: the TPBot car (BLE) is compiled in, off until
  `car_enable`, and costs nothing while off.

Upstream's own apps (AVATAR, AI.AGENT with xiaozhi, ESPNOW, Dance, …) stay on the
launcher. AI.AGENT uses a third-party cloud and is **outside** the platform; its
firmware-update path is closed (`patches/xiaozhi-esp32.patch`).

## The apps it switches between

| App | Repo | Job | Status |
|---|---|---|---|
| **Embody dashboard** | [s-w42-eu-raw](https://github.com/mj41/s-w42-eu-raw) | relay and full remote control: every sensor, every command, camera, mic, speaker, IR, NFC, files | works on the LAN; v0.10.2 at `chan.w42.eu`, with sign-in |
| **Pet** (Tamagotchi) | [s-w42-eu-pet](https://github.com/mj41/s-w42-eu-pet) | a pet for the kids: needs, food via NFC cards, games, naps, routines, parent page with PIN | works on the LAN |
| **sbot cockpit** | [s-w42-eu-sbot](https://github.com/mj41/s-w42-eu-sbot) | Stackchan together with other devices: camera + joystick + head pad + lights; the TPBot car; the sonar safety stop | works on the LAN |
| AI.AGENT (upstream) | — | voice assistant through xiaozhi's cloud | outside the platform |

The same robot, with the same firmware, is a remote-controlled telepresence head,
a kid's pet, or the head of a small car, depending only on which server it joined.
That is principle 22 (switching is simple) in practice.

## The car extension

- **Device:** an ELECFREAKS TPBot (V1 board) with a micro:bit V2 running
  [tpbot-ble](https://github.com/mj41/tpbot-ble) (TinyGo): BLE peripheral, raw sonar,
  line sensors and buttons, motors, headlights, servos, and a 500 ms watchdog.
- **Two adapters, one capability:** first the laptop's `tpbot-bridge`, then
  Stackchan itself over BLE. Both offer the same `car_*` commands and telemetry,
  so sbot did not change when the robot took over.
- **Safety in three layers:** the micro:bit's watchdog, Stackchan stopping the
  motors when its server connection drops, and sbot's sonar safety stop.

## What Stackchan proves for the architecture

- One firmware, many apps: the light-client model works on a real, rich device.
- Raw data first: every feature exposes primary readings; meaning is made on the
  server.
- Hosted devices: a light client can be an adapter for another device (BLE car).
- Optional extensions with no cost to users without the hardware.
- Media only while watched, visible on the device.

## What it does not have yet

- **Trust:** a token per robot, given over USB at setup (a shared one on LAN servers), and
  end-to-end encryption to its enrolled browsers; but no device key, no grants, no scopes.
  The plan is the Stackchan trust design ([Stackchan trust design](stackchan-trust.md)),
  generalized in [architecture §8](../architecture.md#8-trust-identity-and-access-control).
- **App routing:** switching means reconnecting to another server; the node does not
  route yet.
- **One event hub for all apps:** sbot has one (NATS JetStream, embedded); the dashboard and
  the pet keep recent events in memory (and a small state file).
- **Fast car control through the robot:** a car command takes about 0.2–0.3 s through
  Stackchan (`car_*` commands go straight from the WebSocket task to BLE), against 70 ms
  through the laptop bridge.
