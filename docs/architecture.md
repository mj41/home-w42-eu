# Architecture overview

**Status:** draft, 2026-10-02. Describes the target. What exists today, and how it
maps onto this, is in [fit-and-roadmap](fit-and-roadmap.md).

**Everything here serves the [use cases](use-cases.md).** The parts below are means,
not goals: when a part does not help a use case, it waits.

## 1. The picture

```
 ANYWHERE                W42.EU (optional)              HOME
                                                   ┌──────────────────────────────────────────────────┐
 phone / laptop ─ TLS ─► hub: blind relay (TLS   ═══►  HOME NODE                                       │
 away from home          passthrough by name)      │                                                 │
                         rendezvous for devices    │   web/API server ◄──── browsers (cockpit, apps) │
                         encrypted backups         │    ├ device gateway ◄── light clients, adapters │
                                                   │    ├ registry, pairing, policy                  │
                                                   │    ├ apps (cockpit, …) and their APIs           │
                                                   │    └ agent gateway ◄── AI agent harness          │
                                                   │            │  produce / consume                 │
                                                   │            ▼                                    │
                                                   │   EVENT HUB  (log-based, Kafka-like topics)     │
                                                   │            ▲  consume / produce                 │
                                                   │            │                                    │
                                                   │   controller server (async)                     │
                                                   │    ├ loops and controllers                      │
                                                   │    ├ adapters' sources (Home Assistant, router, │
                                                   │    │   calendars) when they run on the node     │
                                                   │    └ archiver, audit, retention                 │
                                                   │                                                 │
                                                   │   stores (one home per record)                  │
                                                   └──────────────────────────────────────────────────┘
       light clients: Stack-chan, micro:bit car, old phones, ESP32 sensors, browser tabs
       adapters:      tpbot-bridge (BLE), Home Assistant, Wi-Fi router, Roomba (serial), cameras (RTSP)
```

Every arrow into the home is an **outbound connection from the client**: devices dial
the node on the LAN, and the node dials the relay. Nothing at home needs an open port.
**At home, nothing goes through `w42.eu`:** devices and browsers on the LAN talk to
the node directly, and some devices never use `w42.eu` at all (§10).

**One home, many repos.** A home is made of the repos its user includes: the node
(growing out of [sbot](https://github.com/mj41/sbot)), device firmware (the [StackChan fork](https://github.com/mj41/StackChan/tree/embody-mj41),
[tpbot-ble](https://github.com/mj41/tpbot-ble)), and app repos that stay separate but join the home when
included ([stackchan-pet](https://github.com/mj41/stackchan-pet), the [stackchan-server](https://github.com/mj41/stackchan-server)
dashboard). All the repos: [The repos today](../README.md#the-repos-today). Each repo is independent; they meet at the
[wire protocol](wire-protocol.md) and the event hub's topics.

## 2. Concepts

| Concept | Meaning |
|---|---|
| **Home** | One household: its owners, people, devices, data and node(s). |
| **Owner** | A person whose key may sign for the home. The root of trust. |
| **Node** | The servers at home: the web/API server, the controller server and the event hub. One machine or several. |
| **Event hub** | The home's log-based message hub: append-only topics, consumers with their own positions, replay from any position within retention. |
| **Device** | Anything that registers through the wire protocol: a robot, a phone, a sensor, a browser, an adapter acting for hardware. Called a *worker* in the protocol. |
| **Capability** | What a device offers: commands it accepts, measurements it reports, events it sends, media it streams. |
| **Hosted device** | A device reached through another one (the TPBot car through Stack-chan's BLE), linked by `with`. |
| **Adapter** | A program that speaks to hardware or a data source in its own way and brings it into the home. |
| **App** | Code with a UI that uses devices: the cockpit, the pet, a dashboard. |
| **Loop** | Code that consumes events and produces commands and decisions, with no UI. |
| **Controller** | A loop with timing and a safety or control role (the sonar safety stop, window shutters). |
| **Home model** | What the home means to its people: people, rooms, places, things, routines, and which devices belong to them (§2.1). |
| **Grant** | A signed statement: this app, loop, user or agent may use these scopes on these devices, until then. |
| **Scope** | A named set of capabilities: `passive.basic`, `active.all`, `media.video`, … |

### 2.1 The home model: meaning, not just data

Raw readings are not what people care about. The **home model** is the node's
description of the home in the words its people use:

- **People:** Ema (kid), Tom (parent), the babysitter (guest, tonight).
- **Places and rooms:** kitchen, Ema's room, the hall, the garden, "school", "work".
- **Things and their devices:** "Ema's phone" (a Wi-Fi client and a GPS source), "the
  robot in the kitchen" (Stack-chan), "the car" (the TPBot), "the west shutters".
- **Routines:** school day, bedtime, "we are away", holidays.

Loops use it to turn raw events into **meaningful events**, each with its cause:

```
 raw:      client_joined {mac_id: 3f9a…}                    (Wi-Fi adapter)
 meaning:  arrived_home {person: Ema, place: home}           (presence loop; cause: the raw event)
 value:    "Ema is home" on Tom's phone; the pet greets Ema  (use case 4)
```

- Meaningful events go into the hub like any other, so further loops, pages and the
  audit can use them ("why did I get this note?" → the raw event that caused it).
- The model is edited by owners in plain forms, and AI agents may propose changes to
  it (a new routine) through the same approval as loops.
- Pages, notifications and the audit show the model's words, never raw ids, unless
  someone asks for the details.

## 3. Light clients

All devices speak the [device wire protocol](wire-protocol.md): an outbound
WebSocket, a `Register` frame with the device's capabilities, then telemetry,
events and commands as JSON, and media as binary messages.

Rules for a light client:

- **Raw data only** (principle 12). The car reports the sonar echo in µs, not
  "obstacle".
- **Stream only while wanted.** Camera, microphone and fast sensors are off until
  the node turns them on, and the device shows it.
- **Safe on its own.** Watchdogs on the device for anything that moves.
- **Capabilities can change.** A device registers again when an extension is enabled
  (Stack-chan's `car_enable`).

## 4. Adapters and data sources

Hardware and services that cannot run a light client join through an adapter. An
adapter either registers devices through the wire protocol (from the node or from
any machine near the hardware), or produces events straight into the event hub when
it runs on the node.

| Adapter | Speaks | Status |
|---|---|---|
| `tpbot-bridge` | BLE to the micro:bit in the TPBot | works (POC) |
| Stack-chan hosting the car | BLE, inside the robot firmware | works (POC) |
| Home Assistant | HA WebSocket API: chosen entities become measurements, events and commands | next: **the main data source for now** |
| Wi-Fi presence | router API (OpenWrt ubus, DHCP leases) | idea |
| Roomba | iRobot Open Interface over serial, through an ESP32 | idea |
| IP cameras | RTSP, frames decoded on the node | idea |
| Calendars | a calendar's secret ICS address or CalDAV; Google or Microsoft APIs only read-only, through the capture station (§4.1) | idea |
| Personal captures | collectors in the capture station on the person's laptop (§4.1) | idea |
| NFC / QR readers | Stack-chan's NFC reader (works), phone cameras, a USB reader | NFC on Stack-chan works |

**Home Assistant is a source, not the brain.** For now it brings the sensors and
devices it already reaches (Zigbee, plugs, contacts, energy, weather, vendor
integrations). Over time, some of its automations move to controllers on the node,
where the algorithms can be smarter than HA supports. The first candidate is the
**window shutters**: custom hardware and software exist already, and they need
control logic beyond HA automations (sun, temperature, wind, presence, time of day).

**Personal data sources** (calendars, later contacts, a shared shopping list) are
adapters too. They turn appointments into coarse events ("school day", "away until
Sunday") under the same scopes and limits as any device data. The node pulls them;
nothing is pushed to the provider.

### 4.1 Personal captures from third-party accounts

People save things all over the place: a family group in Messenger, a bookmarks folder,
saved posts on X or LinkedIn, a task list, a calendar. These are **captures**: things a
person wants to follow up on later. They are valuable inputs (use case 13), and the
most dangerous ones, because a broad token for an account like Google opens mail,
files, photos and every other service behind it.

**Rules for captures:**

- **Never from the node, never with broad access.** Captures are collected only in a
  **capture station**: a secure environment on the person's laptop (a separate OS user
  or a VM, a separate browser profile per service, an outbound allowlist with only that
  service and the home node).
- **Tiny, reviewed collectors.** Each collector is a small Go program with one job
  ("read the `Follow up` folder of my bookmarks"), short enough to review in full,
  pinned by hash, and reviewed again on every change. No collector has general
  account access, and no AI agent runs inside the capture station.
- **The narrowest access that works**, in this order:
  1. **local data** the person already has, with no login at all (the browser's own
     bookmarks file, a downloaded data export);
  2. **a share by hand**: "share to home" from the phone or the browser sends one link
     to the node;
  3. **an API with the narrowest scope** (read-only, one list, one calendar), with the
     token kept only in the capture station.
  A source that needs full account access, or scraping the logged-in website, is not
  collected.
- **Only extracted items leave the station:** `captured {source, kind, title, url,
  saved_at, tags}` to the node through the wire protocol (`class: collector`). No
  credentials, no cookies, no full message histories.
- **Captures are personal data** of the person who saved them: their own store home,
  their own read audit, a local AI model for digests by default (§8.5).

On the node, loops and agents turn captures into value in the person's words: a
weekly "saved for later" digest, "you promised to reply in the family group",
reminders tied to tasks and the calendar.

The same device may be reached by different adapters over time (the car: first the
laptop bridge, then Stack-chan). Apps and loops see the **same capability names**,
so they do not change.

## 5. The web/API server and apps

The **web/API server** is the synchronous half of the node. It grows out of [sbot](https://github.com/mj41/sbot).

- **Device gateway:** the wire protocol endpoint for light clients and adapters.
- **Registry, pairing, policy:** devices, capabilities, people, sessions, grants;
  checks every command against the access model (§8) before it reaches a device.
- **Apps:** each app is a UI plus an API on this server (the cockpit today), or a
  separate app repo included by the user and routed by the node.
- **Producing and consuming:** everything devices send becomes events in the hub;
  apps read live state from the hub and send commands through the gateway.
- **Switching and sharing devices:**
  - Today a device switches apps by connecting to another server from its list.
    Target: devices connect to the node only, and the node routes them to apps.
  - Passive scopes (watching telemetry) can go to many apps at once. Active control
    goes to one app at a time, through a **control lease** that the owner or the
    device can take back. The physical device always wins: a touch on the robot, or
    its power button, overrides any app.

### 5.1 UI sessions: a scan changes the device's UI, then it comes back

Devices with a screen (Stack-chan, an old tablet on the wall, a phone) have a
**default UI**, and **UI sessions** that replace it for a while:

- **A trigger starts a session:** an NFC tag or card, a QR code, a button, a person
  arriving (a meaningful event from the home model), or an app or loop.
- **A session is:** which device, which app and view, for whom (the person whose card
  or phone it was, so their permission entries apply, §8.2a), and **how it ends**.
- **It ends** after a time, after a time without use, on another scan, when the
  person leaves, when the permission it relies on expires, or when someone presses
  "done". Then the device **returns to its default UI** (or to the session it
  interrupted, if sessions were stacked), and the rights go back with it.

```
 default UI ─── Ema's card tapped ──►  Ema's pet session (Ema's permissions)
     ▲                                      │ 10 min without touch, or "done"
     └──────────────────────────────────────┘
 default UI ─── guest scans the robot's QR ──►  guest view on the guest's phone, until 23:00
```

Examples:

| Trigger | On which device | Session | Ends |
|---|---|---|---|
| Ema's NFC card on Stack-chan | the robot | the pet, as Ema; she may feed and play | 10 min without touch |
| Food card on Stack-chan during Ema's session | the robot | the pet eats (inside the session, no switch) | — |
| Guest scans the robot's QR | the guest's phone | guest view: hall camera, lights | the permission's end |
| QR on the washing machine | your phone | the machine's status and history | closing the page |
| A parent arrives home | the kitchen tablet | "welcome home" with the day's events | 5 min |
| The car's safety stop | the cockpit | an "obstacle" overlay | when clear |

- **Who switches:** the node decides and tells the device (target: app routing, §5).
  Today a device switches by connecting to another server from its list; a session
  then is "switch, and switch back later".
- **The device keeps a local fallback:** if the node is unreachable, the device stays
  on, or returns to, its default UI, which never needs the node to show something
  safe (a face, a clock).
- **Physical presence wins:** a touch on the device or its own button can always end
  a session.
- Sessions, their triggers and their ends are events in the hub, and part of the audit
  ("Ema's session on the kitchen robot, 16:02–16:15").

## 6. The event hub

The home's nervous system, memory, audit trail and test data. It works like Kafka
(append-only topics, consumers that keep their own position, replay), sized for one
home.

- **Topics** by kind, for example:
  - `telemetry.<device>`, `events.<device>`: what devices send;
  - `commands.<device>`: what was sent to devices, with the source;
  - `decisions`: what loops and controllers decided, and why;
  - `audit.reads`: who read what, and how much (§8.4);
  - `state.<device>`: the latest value per key (a compacted topic), for fast startup.
- **An event** has: `id` (sortable, e.g. ULID), `ts`, `source` (device, app, loop,
  agent or user), `kind`, `name`, `data`, and `cause` (the id of the event that
  caused it). A command sent by a loop points at the reading that triggered it, so
  "why did the car stop?" has an answer.
- **What goes in:** everything that is not high-rate media. Camera frames, audio and
  100 Hz IMU samples are streamed, not logged. A snapshot or a clip is stored only
  when a loop or an owner asks, with its own retention.
- **Retention per topic:** 7 days hot by default; then sealed to the owner's public
  key, offered for download, deleted (principle 14). Audit topics keep sensitive
  reads longer, for the person concerned (§8.4).
- **Replay:** any consumer can start from an older position. That is how loops are
  tested (§7) and how a new app catches up.
- **Stores** (one home per record, principle 13) hold durable records that are not
  events: people, devices, grants, app data.
- **Several nodes** of one home replicate the hub (each event signed by the node
  that wrote it). Between homes, only what an explicit shared space contains.
- **Implementation:** NATS JetStream, embedded in sbot (since 2026-10-02), listening
  only on localhost with a token, so the controller server can join. Telemetry is
  stored downsampled (one merged message per device per second); every frame is also on
  a live, unstored subject. Kafka-compatible brokers are heavier than one home needs.

## 7. The controller server, loops and AI

### 7.1 Loops and controllers

The **controller server** is the asynchronous half of the node. It runs loops and
controllers as consumers of the event hub.

A loop is: **subscriptions** (topics and filters), **state**, a **handler** (events
in, commands and decisions out), optionally **timers**, and a **grant** (which
devices and scopes it may use).

- **Written in Go**, in the controller server or in a separate repo that the user
  includes. The first one is `frown` in sbot's controller server (the robot looks sad
  while its car is blocked), working on hardware since 2026-10-02.
- **Safety controllers that guard commands stay in the web/API server** (the sbot
  safety stop): a guard must decide before a command reaches the device, and safety must
  not depend on another process running (principle 19). They publish their decisions to
  the hub, where loops use them (`frown` reads the safety stop's decisions).
- **Modes:**
  - `replay`: consume recorded events from an older position, record the commands
    instead of sending them;
  - `shadow`: consume live, write decisions to `decisions`, send nothing;
  - `live`: send commands.
  Every loop goes replay → shadow → live.
- **Commands go through the policy.** A loop's commands pass the same access checks
  as an app's (§8), via the web/API server's device gateway.
- **Kill switch:** each loop can be stopped alone; a home-wide "all loops off" is one
  button and one physical gesture on a robot.
- **WebAssembly later**, for algorithms and extensions that should run on clients
  (in the browser, or on a device). Not needed now.

### 7.2 Controllers and safety

Controllers keep hardware safe or work better than simple rules can: stopping before
a wall, limiting speed near people, window shutters following sun, heat and wind.
They run with priority, their decisions override app commands, and they never
replace the device's own watchdog (principle 19).

Controllers that need low latency run close to the hardware (on the device or its
adapter), and the controller server supervises them. A car command through Stack-chan
takes about 0.8 s today, which is too slow for anything that must react quickly.

### 7.3 AI agents build loops

```
 owner: "when the last phone leaves the Wi-Fi, start the vacuum; stop it if someone comes back"
   │
   ▼
 agent (in an agent harness)  ── reads granted events: Wi-Fi presence, vacuum telemetry
   │  writes the loop + its tests
   ▼
 controller server: replay over the last 7 days ──► "it would have started 3 times: 08:12, 13:40, 17:05"
   │
   ▼
 owner reviews (plain-language summary, the code, the replay) ──► signs the grant
   │
   ▼
 shadow for a day ──► live
```

- **Agents run in any agent harness** that connects to the node as an `ai-agent`
  device through the wire protocol.
- **The agent gateway** on the web/API server offers agents a small tool set:
  describe the home's devices and capabilities, query granted events, write and test
  a loop, run a replay, and ask for approval.
- **What agents cannot do:** sign, run a loop live, widen a grant, see media or
  location unless granted for that task, or keep data after the task.
- **The same path serves controllers** as well as conveniences: an agent can propose
  a better safety stop or shutter algorithm, and it goes through the same replay,
  shadow and approval.

## 8. Trust, identity and access control

### 8.1 Identity

Generalizes the Stack-chan trust design ([design.md](https://github.com/mj41/stackchan-mj/blob/main/docs/design.md) in stackchan-mj) to
every device:

- **Owner key** (ECDSA P-256) signs: device certificates, the node certificate,
  grants, configuration, firmware manifests. A tool (`homectl`, from the planned
  `chanctl`) does the signing; later a passkey on a phone.
- **Device identity:** each device has its own key (in NVS first; in a secure
  element where the hardware has one, e.g. the ESP32-S3 DS peripheral). It proves
  itself with a challenge, not a shared token.
- **People:** owners, family members (accounts on the node, passkeys), guests.
- **Rollback protection:** every signed document has a serial.

### 8.2 The access model: restricted by default

Every request is **(who, through which device, which app or loop, which capability,
on which device)**. It is allowed only if every part allows it:

```
 effective scopes = person's scopes ∩ device-as-proxy's scopes ∩ app's (or loop's) grant ∩ target device's ACL
```

- **Default deny.** No grant means no access. A new device, app, loop or agent
  starts with nothing; a new person starts with what the owner gave their role.
- **Per-device access control.** Each device has its own list: which people, apps
  and loops may use which of its capabilities (e.g. everyone may see the car's
  sonar, only parents may drive it, only the safety controller may stop it without
  a lease). The device enforces it (principle 5), the node enforces it too.
- **Devices as proxies for a person's permissions.** A device can carry a person's
  rights, limited by what the device itself is allowed:
  - a kid's NFC card held to the robot gives the kid's scopes on that robot, for as
    long as the session lasts;
  - a family member's phone carries their scopes when they are away;
  - a wall tablet in the kitchen carries "household" scopes, never a person's
    camera or location rights;
  - a guest who scans a robot's QR gets the guest scopes of that robot only, for
    an hour.
- **Trust levels with limits.** Owner, family member, kid, guest, app, loop and
  agent are trust levels. Each level has default permission entries **and limits**:
  how much it may read, how often it may act, and which media it may ever be granted.
  Trust can be raised by an owner's signature, for a device, a capability and a time
  (§8.2a).

### 8.2a Permissions: who × device × capability × time

The unit of access is a **permission entry**. Scopes such as `active.basic` are only
shortcuts for sets of capabilities; what is stored, signed and checked is the entry.

| Field | Meaning | Examples |
|---|---|---|
| `who` | a person, a role, an app, a loop or an agent | `person:ema`, `role:guest`, `app:cockpit`, `loop:presence` |
| `device` | one device, or a group from the home model | `stackchan-0a1b2c3d4e50`, `tpbot-1a2b`, `room:kitchen`, `kind:camera` |
| `capability` | one command, measurement, event or stream, or a scope shortcut | `car_drive`, `look`, `car_echo_us`, `media.video`, `active.basic` |
| `access` | `see` (data and events), `act` (commands), `stream` (live media) | |
| `time` | **when it is valid**: `for` a duration from when it is granted, or `from`/`until`; optionally a weekly `schedule` | `for: 2h`; `until: 2026-10-03T00:00`; weekdays `15:00–18:00` |
| `via` | through which proxy devices it may be used (optional) | Ema's NFC card, Ema's phone |
| `limits` | rate and history limits (optional, §8.3) | `history: 1d`, `per_hour: 600` |
| `granted_by`, `serial`, signature | who signed it, and rollback protection | an owner's key |

Examples, written in plain words on the page and stored as signed entries:

```yaml
# Ema may drive the car and move the kitchen robot's head, on weekday afternoons.
- who: person:ema
  device: [tpbot-1a2b, stackchan-0a1b2c3d4e50]
  capability: [car_drive, car_stop, look, nod]
  access: act
  time: {schedule: "mon-fri 15:00-18:00"}
  via: [nfc:ema-card, phone:ema]

# The babysitter, tonight only: see the hall camera, switch the lights. Nothing else.
- who: person:babysitter
  device: [kind:light, camera:hall]
  capability: [light_set, media.video]
  access: [act, stream]
  time: {for: 5h}

# The presence loop may see Wi-Fi joins and leaves, never the raw MAC list history beyond a day.
- who: loop:presence
  device: [router:home]
  capability: [client_joined, client_left]
  access: see
  time: {until: 2027-01-01}
  limits: {history: 1d}
```

**How it is checked:**

- A request is allowed only if **an entry that is valid now** matches its `who`,
  `device`, `capability` and `access`, and its `via` if the entry has one, and the
  other parts of the request allow it too (the formula in §8.2). Nothing matches →
  deny.
- **Time is part of every entry.** Entries without an end are only for owners and
  for loops the owner approved; everything given to people other than owners, to
  guests and to agents ends by itself. Owners can hand out "for 30 minutes" with one
  tap.
- **Expiry acts immediately:** when an entry ends, the node closes what it allowed:
  a control lease is taken back, a stream stops, the guest's page shows "your access
  ended".
- **Devices enforce the entries about themselves.** The node sends each device the
  signed entries that name it; commands carry the `who` and `via` of their source
  (wire protocol v2), so the device can check them. A device needs the right time to
  check `time`: until its clock is synced it accepts only owners.
- **Every grant, use and expiry is audited** (§8.4), in the home model's words:
  "the babysitter watched the hall camera for 12 minutes; her access ended at 23:00".

### 8.3 Limits on reading: rate limits and export budgets

Reading is guarded like acting, because scraping is how data leaks:

- **Rate limits per caller and per kind of data**, e.g. an app may read the car's
  telemetry live but at most 1 day of history per hour; an agent may query location
  history only at the precision and range its task needs.
- **Export budgets.** Bulk export (the sealed archive, a camera's clips, a week of
  location) needs an owner or the data's person, and is logged with its size.
- **Anomaly stop.** A caller that suddenly reads far more than usual is paused and
  the owners are told, instead of being allowed to finish.
- **Answers, not dumps.** Agents and apps get the derived answer they asked for
  ("someone was home on Tuesday afternoon") rather than the raw stream, where that
  is enough.

### 8.4 Auditing

- **Every read and every action is logged**, with who, through which device, which
  app, and how much. Reads of sensitive kinds (location, media, calendars, presence)
  are shown to the person they are about.
- **Owners see it in plain words:** "the cockpit app read 3 h of the car's sonar",
  "the vacuum loop read Wi-Fi presence 12 times today".
- The audit log follows the event hub's retention, but sensitive-read records are
  kept long enough for the person to see them (default 30 days, configurable).

### 8.5 AI models and sensitive data

- **Restricted model choice per kind of data.** Media, location, calendars and
  presence go only to models the owner allowed for them, by default a **local model**
  on a home node. A hosted model gets only data its grant names, and preferably
  derived answers instead of raw data.
- Agents work under the same model as apps: scopes, limits, audit, and no standing
  authority.

### 8.6 Physical tokens: NFC and QR

- **QR codes** pair browsers with devices (works today), give guest access, and
  mark places and things. A code never names a host and keeps its secret in the URL
  fragment, so a relay never sees it (§10).
- **NFC tags and cards** identify people (a kid's card), things (a food card for the
  pet, works today) and places (a tag at the door). A tag read is a raw event from
  the reader; what it means is a loop's decision. Cards carrying a person's rights
  are signed by the owner, so a copied UID alone grants nothing.
- **A scan can change a device's UI** for a while, with the scanned person's rights,
  and the device returns to its default UI afterwards (§5.1).

### 8.7 Delegated actions: secrets stay in their perimeter

Some actions need a secret: signing a firmware release, paying by card, logging in to a
third-party account, signing a permission. The secret lives in one **security
perimeter** and never leaves it. Every other device that needs the action **asks a
device inside the perimeter to do it**, and gets back only the result.

**Perimeters** (each one a place where secrets live, and nothing else):

| Perimeter | Holds | Does on request |
|---|---|---|
| Owner signing station (the owner's laptop, a YubiKey, later a passkey on a phone) | the owner key | sign firmware manifests, permission entries, device certificates, node certificates |
| Capture station (§4.1) | third-party account tokens | collect from one part of an account |
| A person's phone (its secure element, its wallet or bank app) | payment cards, passkeys | pay, prove the person's identity |
| A release station (a reviewed build machine) | release signing keys, deploy credentials | publish a reviewed release to a server |

**The flow:**

```
 robot / app / loop                       holder device (inside the perimeter)
   │ action_request {id, action, exact          │
   │   parameters, why, expires}                │
   ├──────────────── via the node ──────────────►│ shows it in plain words:
   │                                            │ "Pay 18.40 EUR to Pizza Roma for the kids' dinner?"
   │                                            │ "Publish sbot v0.3.1 (sha256 3f9a…) to the home server?"
   │                                            │ the person checks and approves locally
   │                                            │ (PIN, fingerprint, YubiKey touch)
   │                                            │ the holder performs the action itself
   │◄─────────────── action_result {id, ok,  ────┤
   │     receipt / signature / reason}           │
```

**Rules:**

- **The secret never leaves the perimeter**, not even as a short-lived token. The
  requester gets an outcome (paid, receipt id; signed, the signature; published, the
  version) that cannot be reused for another action.
- **Exactly what will happen is shown,** and is exactly what is done: the amount and the
  merchant, the file hash and the target. A changed parameter is a new request.
- **Only allowed requesters, only allowed actions:** a permission entry (§8.2a) says
  which devices, apps or loops may request which actions, from which holders, how often
  and up to how much ("the kitchen robot may request food orders up to 30 EUR, at most
  twice a day").
- **Requests expire and cannot be replayed** (an id, a nonce, a deadline).
- **The holder may say no without saying why,** and silence is a no.
- **Card data never touches an insecure device.** The robot, the node and any AI agent
  never see a card number; a payment happens only on the person's phone, inside its own
  payment perimeter, or not at all.
- **Everything is audited** on both sides, in plain words: who asked, what for, who
  approved, what happened.

## 9. Interfaces

- **Browser** on the LAN (a PWA served by the web/API server), or from anywhere
  through the relay. The sbot cockpit is the first: video, joystick, head pad and lights on
  one screen.
- **Devices themselves:** a robot's face, voice, LEDs, touch and NFC are an
  interface to the home, not only to the robot's own app.
- **Physical tokens:** QR codes and NFC tags for pairing, guests, people and
  triggers (§8.6).
- **Notifications:** from loops, delivered by the node to the owners' devices
  (through the relay when away), never through a vendor push service that sees the
  content.

## 10. Connection paths: local first, w42.eu only as a hub

**`w42.eu` is only a hub.** It helps devices and browsers find the home and reach it
from outside. It is never in the path when a direct, local path exists, and a home
can run with no `w42.eu` at all.

### 10.1 At home: straight to the node

- **Devices and browsers on the home network connect directly to the node**, by a
  local name or address: no detour through the internet, latency in milliseconds,
  and no outside party sees even the metadata.
- **How a device finds the node,** in order:
  1. the local endpoints in its configuration (a local alias such as `home.lan`, or
     an IP);
  2. an owner-signed **endpoint announcement** (the node's current LAN URLs), which
     the device may get from the node itself or from the rendezvous;
  3. a local discovery name (mDNS), later.
- **One name inside and outside, without detours:** the node's public hostname
  (`<node>.n.w42.eu`) can resolve to the node's LAN address inside the home
  (split-horizon DNS on the home router). Then the same URL and the same certificate
  work at home (direct) and away (through the relay), and a browser never needs to
  switch. Some routers block public names that resolve to private addresses (DNS
  rebinding protection); that one name must be allowed.
- **Without a public name:** a local alias with a certificate from the owner's CA
  (devices carry the owner key anyway); browsers need the owner CA installed once,
  or accept plain `http` on the LAN for low-risk pages.
- **Switching:** a device that loses the local path (a phone leaving home) falls back
  to the hub only if its policy allows it (§10.3), and switches back to local as soon
  as the local endpoint answers again. Stack-chan's server list is the first form of
  this.

### 10.2 Away from home: through the hub

- **Browsers** reach the node through the `w42.eu` **blind relay**:
  - each node has its own hostname (`<node>.n.w42.eu`);
  - the relay routes connections by that name **without decrypting** (TLS
    passthrough), into the node's outbound tunnel;
  - TLS ends on the node, which gets its own certificate (Let's Encrypt, TLS-ALPN-01
    through the same passthrough), and all web code comes from the node;
  - the node watches the public Certificate Transparency logs for its name, so a
    certificate it did not request is detected.
- **QR codes** that point through `w42.eu` keep their secret in the URL fragment,
  which browsers never send to the server and keep across redirects.
- **Devices outside the home** (a phone with GPS, a car's camera) reach the node the
  same way. The rendezvous (`chan.w42.eu`) only hands out owner-signed endpoint
  announcements; it cannot point a device anywhere the owner did not sign.
- **Direct when possible, later:** two parties that can reach each other directly
  (for example over a VPN the owner runs) use that instead of the relay.

### 10.3 Relay policy per device

Each device has a **relay policy**, part of its owner-signed configuration:

| Policy | Meaning | Default for |
|---|---|---|
| `never` | LAN only. The device never connects to `w42.eu`, not even for rendezvous. | cameras, microphones, devices in kids' rooms, anything the owner marks private |
| `when-away` | local first; the hub only when the local node cannot be reached | phones, laptops, devices that travel |
| `always` | may use the hub even at home | only for testing |

- A device with `never` simply does not work outside the home. For some devices that
  is the point: their data cannot leave the house by any path.
- The node enforces the same policy for streams: media from a `never` device is not
  sent through the relay, even to an owner who is away, unless the owner changes the
  device's policy.
- The policy is shown on the device's page, and changes to it are audited.

### 10.4 Many machines and many homes

- **More than one machine per home** (a Pi always on, a laptop with a GPU for vision
  and AI): one primary; others join with owner-signed node certificates and
  replicate the event hub, all on the LAN.
- **Between homes:** nothing by default. Shared spaces (a game between two families)
  are explicit, with their own stores and grants.

## 11. Privacy by kind of data

| Data | Default |
|---|---|
| Sensor readings, events, commands | event hub, 7 days hot |
| Camera frames, audio | streamed only while watched; not stored |
| Snapshots, clips | stored only when asked; short retention; per camera |
| Location (phone GPS) | to the node only, end to end; coarse by default; history only if the owner turns it on |
| Wi-Fi presence | device MACs pseudonymized; only "known devices of this home" are named |
| Car plates (camera OCR) | processed on the node; only plates on the owner's list are named; others kept as hashes, briefly |
| Calendars | read by the node; appointments become coarse events ("busy", "away"); details only for the calendar's owner |
| Agent context | only what the task's grant covers; deleted after the task |

Every kind has an owner-visible switch and a plain-language page that says what is
kept, where, and for how long.

## 12. Open questions

- **Event hub:** JetStream embedded in sbot works; when should it become its own
  process? Signatures per event or per batch? Is one stored telemetry message per second
  right for every device (a 70-field Stack-chan frame is ~1 KB: ~86 MB a day)?
- **App routing:** how separate app repos join the node: as their own servers behind
  the node's routing, or as packages built into the web/API server.
- **Capability descriptors:** extend `Register`, or a separate document per device
  model (wire protocol v2)?
- **Home Assistant:** which automations move to controllers first, after the window
  shutters?
- **Rate limits:** fixed per trust level, or learned per caller with an owner-set
  ceiling?
- **Permission entries on small devices:** how many entries can a micro:bit or an
  ESP32 hold and check, and does an adapter check them for devices that cannot?
- **NFC cards for people:** which cards can hold an owner signature (NTAG 424 DNA has
  SUN authentication; plain NTAG21x only a UID and data)?
- **Real-time:** which controllers must run on the device or its adapter, given Wi-Fi
  and BLE latency?
- **Local name:** split-horizon DNS for `<node>.n.w42.eu`, an owner-CA certificate for a
  local alias, or both? Which home routers allow it?
