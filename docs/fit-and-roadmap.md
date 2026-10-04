# How the existing work fits, and what comes next

**Status:** 2026-10-04.

## 1. The map

The repos and their roles: [The repos today](../README.md#the-repos-today) in the README.

**What the POCs proved:** light clients on real hardware (Stackchan, micro:bit),
adapters that can be swapped without touching apps (bridge → robot), app switching
by server list, raw-data-first telemetry, and safety in layers (device watchdog,
link-loss stop, node controller).

## 2. The gaps

| Gap | Today | Target |
|---|---|---|
| One home node | three app servers, each with its own pairing and sessions | `sbot` as the web/API server; apps routed by the node; separate app repos included by the user |
| Home Assistant | not connected | the main data source, through an adapter |
| Trust | bearer tokens: one shared per LAN server; on w42.eu one per robot per app, issued by sm.w42.eu | owner key, device keys, grants, scopes |
| Access control | anyone paired with a robot can do everything; on raw.sa.w42.eu robots are private to their owner by default, and tiers set limits; loops in sbot need grants | default deny, per-device ACLs, devices as proxies, limits, read audit |
| AI building loops | agents write code in repos, by hand | agent gateway: query, write, replay, ask approval |
| Capability descriptors | lists of command and measurement names | wire protocol v2: arguments, units, scopes, safety class |
| Connection paths | devices connect to whichever server is in their list; `raw.sa.w42.eu` relays for signed-in owners, TLS ending before the server, end-to-end encryption available per robot | local first: direct on the LAN by alias or IP; `w42.eu` only as a hub when away (blind relay, TLS on the node); per-device relay policy, LAN-only devices |
| Delegated actions | secrets sit where they are used (a robot's token on the robot); releases are signed on the owner's laptop | secrets stay in their perimeter (signing station, capture station, phone); others send action requests; the person approves on the holder |
| Personal captures | none | capture station with tiny reviewed collectors; captures in the person's store; digests |
| Latency through Stackchan | about 0.2–0.3 s per car command | ~0.1 s |

## 3. Roadmap

The order follows the [use cases](use-cases.md): each stage makes one or more of them
work for real people, leaves a working system, and is proved on our own hardware.

1. **Done: commit the POCs** (tpbot-ble, sbot, stackchan, the notes, these two repos).
2. **Done: event hub and controller server.** The hub (NATS JetStream, embedded in sbot)
   holds telemetry, events, commands and decisions. The controller server runs loops with
   the loop interface and the three modes (replay, shadow, live); the first loop links two
   devices (`frown`: Stackchan frowns while the car is blocked). The safety stop stays in
   sbot, whose command guard must decide before a command reaches the car. *(use cases 3, 10)*
3. **Home Assistant as the data source, and the home model v1.** People, rooms and
   which devices belong to them, so events can say "the kitchen" and "Ema". An
   adapter for a first set of entities
   (door contacts, temperatures, plugs, weather), events in the hub, a loop that uses
   them ("look at the door"). *(use cases 5, 6)*
4. **Window shutters controller.** Bring the existing custom shutter hardware and
   software in as a device, and run its algorithm as a controller on the node, with
   HA data as input. The first HA automation replaced by something smarter. *(use case 5)*
5. **Trust**, generalizing the Stackchan design: owner key and `homectl`, device keys
   for Stackchan and the micro:bit (BLE pairing or an allowlist), grants and scopes
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

Running alongside: [device setup](device-setup.md) §7, [end-to-end encryption](e2ee.md),
[accounts](accounts.md), [independent rebuild](independent-rebuild.md).

Ideas that are not on the roadmap yet live in [home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas).

## 4. Where it stands

The POCs fit the **device side** well (light clients, raw data, optional extensions, safety
on the device), and the **node side** has started: sbot embeds the event hub (JetStream) and
keeps the safety stop; the controller server runs loops with replay / shadow / live and loop
grants (none by default), and `frown` is live. On w42.eu the manager (sm.w42.eu) has sign-in,
one-click setup and a token per robot per app; its apps are the raw dashboard (raw.sa.w42.eu:
private robots, tiers, end-to-end encryption available) and the pet (pet.sa.w42.eu);
releases are reproducible and approved in `mj41cz-approved`.

Still open against the [principles](principles.md) and the [architecture](architecture.md):

- **Trust (principles 3, 5):** bearer tokens; no device keys; commands carry no `who`, so
  devices cannot enforce permissions; the robot keeps its Wi-Fi password in plain text in NVS.
- **Access (principles 6, 7):** anyone paired with a robot may use all its commands and see
  all its data; no permission entries, no read audit.
- **Local name and TLS (architecture §10.1):** plain `http`/`ws` on the LAN; no local name, so
  no microphone in browsers (they need HTTPS).
- **Hub, not cloud (principle 4):** at raw.sa.w42.eu TLS ends before the server; only robots
  with end-to-end encryption on are hidden from it.
- **Local first (principle 11):** the pet's spoken lines go to Microsoft's Edge text-to-speech
  (espeak-ng is the local fallback); a local voice (Piper) would remove it.
- **One node:** the pet and the dashboard keep their own pairing and sessions, not yet routed
  by the node.

## 5. What to do next

**Value (stages 3–4):** Home Assistant as the data source with the home model v1,
and the window shutters controller (use case 5): the first use case that helps the
whole family every day.

**Alongside:** end-to-end encryption on for raw.sa.w42.eu ([e2ee](e2ee.md) §9, step 4), and
the flasher as its own part ([device setup](device-setup.md) §7).

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
