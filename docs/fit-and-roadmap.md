# How the existing work fits, and what comes next

**Status:** 2026-10-02.

## 1. The map

| Repo | Part of the [architecture](architecture.md) | State |
|---|---|---|
| `sbot` | **grows into the home node**: today the web/API server with the cockpit app, many devices, hosted-device links (`with`), the first controller (safety stop), a small state file | works on the LAN |
| `stackchan` (firmware fork, branch `embody-mj41`) | light client (§3); hosts the car over BLE (§4) | works on the LAN |
| `stackchan-server` | the Go implementation of the [wire protocol](wire-protocol.md) (`wire` package); a separate dashboard app (§5); later the rendezvous role (§10) | works; v0.1.0 at `chan.w42.eu` |
| `stackchan-pet` | a separate app repo, part of a home when included (§5) | works on the LAN |
| `tpbot-ble` | micro:bit firmware (light client), the `tpbot` tool, the `tpbot-bridge` adapter (§4) | works |
| `stackchan-mj` | Stack-chan workspace notes, build and run scripts, and the trust design (`docs/design.md`) that §8 generalizes | design for trust |

**What the POCs proved:** light clients on real hardware (Stack-chan, micro:bit),
adapters that can be swapped without touching apps (bridge → robot), app switching
by server list, raw-data-first telemetry, and safety in layers (device watchdog,
link-loss stop, node controller).

## 2. The gaps

| Gap | Today | Target |
|---|---|---|
| One home node | three app servers, each with its own pairing and sessions | `sbot` as the web/API server; apps routed by the node; separate app repos included by the user |
| Event hub | recent events in memory per server | log-based hub (Kafka-like), topics with retention, replay |
| Controller server | one controller inside sbot's web server | async controller server consuming the hub: replay, shadow, live |
| Home Assistant | not connected | the main data source, through an adapter |
| Trust | one shared bearer token per server | owner key, device keys, grants, scopes |
| Access control | anyone paired with a robot can do everything | default deny, per-device ACLs, devices as proxies, limits, read audit |
| AI building loops | agents write code in repos, by hand | agent gateway: query, write, replay, ask approval |
| Capability descriptors | lists of command and measurement names | wire protocol v2: arguments, units, scopes, safety class |
| Connection paths | devices connect to whichever server is in their list; `chan.w42.eu` relays plain WebSocket for one deployment | local first: direct on the LAN by alias or IP; `w42.eu` only as a hub when away (blind relay, TLS on the node); per-device relay policy, LAN-only devices |
| Delegated actions | secrets sit where they are used (a shared token on the robot) | secrets stay in their perimeter (signing station, capture station, phone); others send action requests; the person approves on the holder |
| Personal captures | none | capture station with tiny reviewed collectors; captures in the person's store; digests |
| Latency through Stack-chan | ~0.8 s per car command | ~0.1 s |

## 3. Roadmap

The order follows the [use cases](use-cases.md): each stage makes one or more of them
work for real people, leaves a working system, and is proved on our own hardware.

1. **Commit the POCs** (tpbot-ble, sbot, stackchan, stackchan-mj, these two repos).
2. **Event hub and controller server.** Add the hub (NATS JetStream first) next to
   sbot; sbot produces telemetry, events, commands and decisions into it. Start the
   controller server, and move the safety stop there with the loop interface and the
   three modes (replay, shadow, live). Add a second loop that links two devices
   (Stack-chan frowns while the car is blocked). *(use cases 3, 10)*
3. **Home Assistant as the data source, and the home model v1.** People, rooms and
   which devices belong to them, so events can say "the kitchen" and "Ema". An
   adapter for a first set of entities
   (door contacts, temperatures, plugs, weather), events in the hub, a loop that uses
   them ("look at the door"). *(use cases 5, 6)*
4. **Window shutters controller.** Bring the existing custom shutter hardware and
   software in as a device, and run its algorithm as a controller on the node, with
   HA data as input. The first HA automation replaced by something smarter. *(use case 5)*
5. **Trust**, generalizing the Stack-chan design: owner key and `homectl`, device keys
   for Stack-chan and the micro:bit (BLE pairing or an allowlist), grants and scopes
   enforced on the device and the node. Drop the shared token. The owner's signing
   station is the first security perimeter, and signing is the first delegated action:
   a release or a permission is prepared elsewhere and signed there after the owner
   checks it. *(use cases 9, 11, 14)*
6. **Access control v1:** per-device ACLs, people and roles, NFC cards as proxies for
   kids, guest QR scopes, read audit, rate limits on history reads. *(use cases 4, 9, 11)*
7. **More adapters:** Wi-Fi presence (router), calendars (ICS/CalDAV). *(use cases 4, 6, 8)*
8. **Agent gateway:** the first AI-written loop goes replay → shadow → live with an
   owner's approval. *(use case 10)*
9. **Old phone as a device:** a light client (web page first), starting with camera,
   GPS (private, audited) and screen. *(use cases 7, 12)*
10. **Local first, hub when away.** A local name for the node (split-horizon DNS or an
    owner-CA alias), endpoint announcements and per-device relay policies (cameras and
    microphones LAN only by default), then the blind relay so the cockpit works away
    from home without `w42.eu` seeing anything. *(use cases 1, 11)*
11. **Personal captures.** A capture station on the laptop with the first collectors
    that need no account access (one bookmarks folder from the browser's local file,
    "share to home" from the phone), captures in the person's store, and a weekly
    digest. *(use case 13)*

Ideas that are not on the roadmap yet live in the `home-w42-eu-ideas` repo.

## 4. Review against the design (2026-10-02)

What each project does today, checked against the [principles](principles.md) and the
[architecture](architecture.md). ✓ fits, ✗ gap.

### sbot (grows into the home node)
- ✓ Wire protocol device gateway, pairing, SSE, camera relay, snapshots; linked
  devices (`with`); the cockpit; the sonar safety stop; server-only commands blocked.
- ✗ **Access (principles 6, 7):** any session paired with a robot may use every one of
  its commands and see all its data; no permission entries, no time limits, no read
  audit. Any worker with the shared token can claim `with` to any robot.
- ✗ **Meaning (principle 2):** pages and events show raw ids
  (`stackchan-0a1b2c3d4e50`, `tpbot-1a2b`), not "the kitchen robot", "the car".
- ✗ **Event hub (§6):** 20 recent events per device in memory; nothing persistent, no
  causes. The safety stop runs inside the web server, not in a controller server.
- ✗ **Local name and TLS (§10.1):** plain `http`/`ws` on the LAN; no local name, so no
  microphone in browsers (they need HTTPS).

### stackchan (firmware, Embody Mode)
- ✓ Light client: raw data, media only while watched with a LIVE badge, a local
  default UI (the face), a server list (the first form of app switching and sessions),
  the car as an optional extension, safe stops.
- ✗ **Trust (principles 3, 5):** one shared token compiled in; no device key; commands
  carry no `who`, so the robot cannot enforce permissions; the Wi-Fi password is plain
  text in NVS.
- ✗ **Use case 3:** a car command takes ~0.8 s through the robot (commands wait in the
  app loop).

### tpbot-ble (micro:bit firmware, tool, bridge)
- ✓ Raw state, a 500 ms watchdog on the device (principle 19), the `car_*` capability
  shared by the bridge and the robot.
- ✗ **Restricted by default (principle 6):** the BLE service is open; any phone nearby
  can connect and drive the car.
- ✗ The bridge uses the shared token, and does not release BlueZ's link when it exits.

### stackchan-server (wire package, dashboard, `chan.w42.eu`)
- ✓ The Go implementation of the wire protocol; the full-control dashboard.
- ✗ **Hub, not cloud (principle 4, §10):** at `chan.w42.eu` TLS ends at the cluster
  gateway, so the cloud sees all traffic. That is a relay, not yet the blind hub.
- ✗ Its `wire` package comment still points at another project as the reference; it
  should point at this repo's [wire protocol](wire-protocol.md).

### stackchan-pet (separate app, included by the user)
- ✓ An app in its own repo, NFC food cards, routines, a PIN-protected parent page: real
  use case 2.
- ✗ **Local first (principle 11):** spoken lines go to Microsoft's Edge text-to-speech
  service (cached on disk; espeak-ng is the local fallback). The lines are fixed phrases
  with foods and scores, so the risk is low, but it is a cloud in the data path. A local
  voice (Piper) would remove it.
- ✗ Its own pairing and sessions, separate from sbot's (to be routed by the node).

### Overall
The POCs fit the **device side** well (light clients, raw data, optional extensions,
safety on the device). The gaps are all on the **node side**: no event hub, no home
model, no permissions, and one shared token everywhere. Those are what to build next.

### Done since the review (2026-10-02)
- ✓ **Car latency through Stack-chan:** `car_*` commands go from the WebSocket's receive
  task straight to BLE. Command to telemetry echo 185–285 ms, with or without the camera
  streaming (was ~0.8 s with the camera on).
- ✓ **BLE allowlist** on the micro:bit (tpbot-ble 0.3.x): only listed centrals; a refused
  central gets no commands taken and no state, and is dropped. Tested with a dummy list.
- ✓ **Names and rooms** in sbot, shown instead of ids.
- ✓ The `wire` package and stackchan-server docs point at this repo's protocol.
- ✓ `tpbot-bridge` releases BlueZ's link when it is stopped.
- ✓ **Stage 2:** the event hub (JetStream embedded in sbot: telemetry, events, commands
  with source and cause, decisions), the controller server with replay / shadow / live,
  loop grants (none by default), and the first loop, `frown`. Tested end to end with fake
  devices; running on the real robot in shadow mode.
- Found and fixed on the way: the micro:bit could report a stale sonar distance after
  the sonar was switched off during a measurement (tpbot-ble 0.3.1).
- **The first shadow run paid off:** a hand in front of the sonar made the safety stop
  flap on and off (close up, every other echo is missed: 8.8 cm, 0, 8.7 cm, 0 …). The
  frown loop in shadow mode recorded 76 would-be faces in 30 s before it sent anything.
  Fixed in sbot: a missed echo no longer ends an episode. Then `frown` went live.

## 5. What to do next

**First, small fixes that remove known gaps (done 2026-10-02, see above):**
1. **Car latency through Stack-chan:** run `car_*` commands straight from the client
   callback, not the app loop (use case 3).
2. **BLE allowlist on the micro:bit:** accept only listed central addresses (Stack-chan,
   the laptop), set with a button press on the micro:bit (principle 6).
3. **Names in sbot:** a first, tiny home model (device → name, room), shown everywhere
   instead of ids (principle 2).
4. **`stackchan-server` `wire` package comment** points at this repo's protocol.
5. **`tpbot-bridge` disconnects BlueZ** on exit.

**Then the foundation (stage 2, done 2026-10-02):** the event hub (NATS JetStream) next to sbot, the
controller server, the safety stop moved there with replay / shadow / live, and the
first loop between two devices ("the robot frowns while the car is blocked"). It proves
the node's architecture on hardware that already works (use cases 3, 10).

**Then value (stages 3–4):** Home Assistant as the data source with the home model v1,
and the window shutters controller (use case 5): the first use case that helps the
whole family every day.

## 6. Decisions made (2026-10-02)

- **The home node grows out of `sbot`.** Some repos stay separate (apps, firmware,
  adapters) and are part of a home when its user includes them.
- **Home Assistant is a data source** for now. Some of its automations get replaced
  by controllers later, where they need smarter algorithms than HA supports (window
  shutters first).
- **Two servers around an event hub:** a web/API server, and an async controller
  server consuming a Kafka-like, log-based event hub. WebAssembly maybe later, for
  algorithms and extensions on clients; not needed now.
- **The repos are independent.** They meet at the wire protocol defined in this repo
  and at the hub's topics.
