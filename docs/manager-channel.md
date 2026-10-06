# The robot's manager: one primary, sm.w42.eu upstream

**Status:** proposed 2026-10-05; built and tested 2026-10-06 (§11). Changes how a Stackchan manager (a home's own
s-w42-eu-manager, or sm.w42.eu) reaches a robot. Builds on [sm-ux.md](sm-ux.md) and the node
and relay of [architecture.md](architecture.md); the frames are in
[wire-protocol.md](wire-protocol.md).

## 1. Why

Today a manager reaches a robot only **through the app the robot is on**: the app asks the
manager about the robot every 15 s (robot-auth) and passes on the manager's signed app list. A
robot may have several managers, each with its key on the robot. On 2026-10-05 this showed its
limits on a real robot:

- **A manager could not reach the robot while it was on another manager's app.** The robot was
  on Raw data at raw.sa.w42.eu (sm.w42.eu's); the home manager showed it offline and offered no
  Switch; sm.w42.eu could not switch it to a home app.
- **Slow and blind.** A switch from the page arrived within 15 s; then the page said
  "✓ On the robot" while the robot still waited for a Yes on its screen. The owner clicked
  twice, nothing seemed to happen, the question ran out unseen.
- **The robot hung** (a question opened without the screen lock). Nothing could reach it: its
  app connection was dead with it, the manager knew only "not seen for a while".
- **Apps carry the managers' business:** the ManagedApps relay, AppsVersion frames, the
  `apps_ver` label, the polling. Every new app must do it right.

## 2. The design in short

**Principles** ([principles](principles.md) 22): apps are separate; USB or the manager delivers
and removes them; the manager is optional (off on the robot or by the manager itself, on again
only on the robot or over USB); switching is always possible on the robot's QR screen.

```
 ANYWHERE                     W42.EU                          HOME (behind NAT)
 phone, laptop ── page ─►  sm.w42.eu  ◄═══ link (out) ═══  home manager (primary)
                           (read-only shadow,                  │ ▲
                            or full control                    │ │  manager channel (out
                            through the home)                  ▼ │  from the robot, ws on LAN)
                                                              robot ── app connection ──► an app
                                                                       (home Pet, raw.sa.w42.eu, …)
```

- **A robot has one manager at a time, its primary,** chosen at the USB setup: the home manager
  when the home has one, else sm.w42.eu. Only the primary changes the robot. The setup may also
  give a second manager (sm.w42.eu for a home robot) that the person at the robot can make the
  primary on the robot's **Manager screen**, if the setup allowed it (S16).
- **The robot keeps a connection to its primary** (the manager channel), apart from its app
  connection: out from the robot, its own task, up first at boot, kept across app switches.
- **The home manager links up to sm.w42.eu** (out, so NAT is no problem), once linked to the
  owner's account; an account may link several homes. Over the link sm.w42.eu shows the home's
  robots live and issues the tokens of its own apps when the home asks. The owner chooses per
  home (S3):
  - **read-only shadow** (the default): sm.w42.eu changes nothing; at the top of its page
    **Manage at home** leads to the home manager's own page;
  - **full control through the home:** sm.w42.eu's page works like the home's; its requests go
    down the link and the home manager, the only one with its key on the robot, signs them and
    logs them as "from sm.w42.eu" (S15).
- **The robot shows its manager** on a Manager screen (next to the QR screen): which manager,
  connected or not, the home manager's address as a link and a QR code (S14).
- **sm.w42.eu is primary only for robots in homes without a manager.** Then the robot's
  channel goes to sm.w42.eu directly, and its page is a full manager.
- **Apps keep only what is theirs:** checking robot tokens with the manager that issued them
  (robot-auth) and reporting paired browsers. No relaying, no polling.

### 2.1 What the pages and the robot say about connections

Two connections matter to a person: **robot ↔ its manager** (the channel) and **home ↔
sm.w42.eu** (the link). Every place names both ends and says connected or not:

| Where | Connected | Not connected |
|---|---|---|
| a robot's card on its manager | "● connected · on Pet" | "not connected · last seen 5 min ago · on Pet" |
| a home's robot on sm.w42.eu | "● connected to home on laptop · on Pet" | "not connected to home on laptop · last seen …" |
| the home's link card | "This home is linked to sm.w42.eu, connected", and what sm.w42.eu may do (shows your robots read-only, or may change them; the home signs) | "…, not connected now (the error)" |
| the home on sm.w42.eu | "this home is connected · read-only here: changes at home" | "this home is not connected, last seen …" |
| the robot's Manager screen | which manager, connected, its page as a link and a QR code | "not connected" and the last error; or "off" |

- **After an hour away** the card adds why that may be: the robot is off, away from its Wi-Fi,
  or was set up with another manager over USB while not connected here. It keeps the robot as
  it was last seen, and what it knew is marked so: "✓ On the robot when it was last seen".
- **Apps of a linked manager** carry its name on their chips at home ("Raw data sm.w42.eu"), so
  two apps with the same name are told apart.
- **"Watching now"** under Paired browsers comes only from an app report under a minute old.
- **A robot set up with another manager over USB** tells the old one on its way out (`Leaving`,
  S19): the old card then says "Set up with ‹to› over USB …" instead of the away note, and
  changes there are refused. Only a robot whose channel was up can say so; otherwise the away
  note stays the only hint.

## 3. Use cases

| # | Who | Wants | How (sequence) |
|---|---|---|---|
| U1 | owner with a home manager | set a new robot up at home | S1 |
| U2 | owner without a home node | set a robot up with sm.w42.eu only | S2 |
| U3 | owner | see the home's robots on sm.w42.eu from anywhere | S3, S7 |
| U4 | owner | give a home robot an sm.w42.eu app (Raw data at raw.sa.w42.eu) | S4 |
| U5 | owner | change the robot's apps or start app | S5 |
| U6 | owner at the robot's card | open one of its apps and the robot comes along, in seconds, seeing the question and the answer | S6 |
| U7 | person at the robot | switch apps on the robot's screen; the pages follow | S7 |
| U8 | owner | remove a paired browser (a lost phone), on any app, and know it is out | S8 |
| U9 | owner | get a hung robot back without the cable | S9 |
| U10 | home | the internet is down: everything at home keeps working | S10 |
| U11 | owner | the home manager is off: the robot keeps running its apps | S11 |
| U12 | owner | move a robot to another primary (a new home node, a new owner) | S12 |
| U13 | later | firmware updates and settings over the air | S13 |
| U14 | owner on sm.w42.eu or at the robot | find the home manager's page (its local address) | S14 |
| U15 | owner away from home | change a home robot from sm.w42.eu, if the home allows it | S15 |
| U16 | person at the robot | make the other manager the primary (the robot goes to a cottage, the home node is gone), if the USB setup allowed it | S16 |
| U17 | owner with a cottage | see and manage the robots of several homes on one sm.w42.eu account | S3 |
| U18 | owner who wants no remote control | turn the manager off: apps only over USB, switching on the robot | S17 |
| U19 | owner with a phone at home | open the home manager's page on the phone (no sign-in at home) | S18 |
| U20 | owner who sets the robot up with another manager over USB (moving it, or trying sm.w42.eu alone) | the old manager's page says where it went, instead of a robot that is just away | S19 |

## 4. Sequences

`page` is the manager's page in a browser; `→` a message; **signed** means signed with the
primary's key and checked by the robot.

**S1. First setup with the home manager (USB)**
1. page (home) → robot over USB: `hello`; the robot is new.
2. The owner picks apps and the start app; the page → home manager: setup.
3. home manager → page: the app list with a token per app, its key, its channel URL
   (`ws://<home>:8790/robot`) and a **channel token** for this robot (the manager keeps its
   SHA-256).
4. page → robot over USB: `provision` with all of it; the robot asks "Set up by home on
   laptop?" on its screen; Yes.
5. robot → home manager: opens the channel with its token, `Hello {firmware, apps, app}`.
6. robot → its start app: connects. The page shows the robot live.

**S2. First setup with sm.w42.eu only**
As S1 with sm.w42.eu as the primary: the channel goes to `wss://sm.w42.eu/robot`, and
sm.w42.eu's page is the full manager.

**S3. Linking the home manager to sm.w42.eu (once per home)**
1. page (home) → **Link to sm.w42.eu**, with two choices: **sm.w42.eu may change the robots**
   (off: a read-only shadow) and **show this manager's local address there** (off).
2. The browser signs in on sm.w42.eu and approves "Link home on laptop to your account".
3. sm.w42.eu → page → home manager: a **link token** (one per home manager; sm.w42.eu keeps its
   hash). An account may hold several (home, cottage).
4. home manager → sm.w42.eu: opens the link (`wss://sm.w42.eu/link`) with the token; `Home
   {name, mode, local_url?}`, then `Robots [{id, name, state}]` and every state change after.
5. sm.w42.eu shows those robots on the account's page, grouped by home: live, "managed by home
   on laptop". The choices change only on the home's page.

**S4. An sm.w42.eu app for a home robot**
1. page (home): **+ add** lists the home's catalog and, while linked, sm.w42.eu's apps.
2. home manager → sm.w42.eu over the link: `Grant {robot, app: "raw"}`.
3. sm.w42.eu: creates the robot's token for raw.sa.w42.eu (keeps its hash, owner = the
   account) → home manager: `{url, token}`.
4. home manager → robot: **signed** `Apps` (with the new app and its token), as in S5.
5. Later raw.sa.w42.eu checks the robot's token with sm.w42.eu (robot-auth) as today.

**S5. Changing apps**
1. page (primary) → manager: the new apps and start app; the version + 1.
2. manager → robot over the channel: **signed** `Apps {version, servers, pin}`.
3. robot: checks, replaces its list; a new start app is asked on its screen when `ask_pin`;
   → manager: `State {apps_version, question?}`; the page shows it, then "on the robot ✓".

**S6. Switching from the page**
1. page (primary): the owner opens Raw data; the robot is on Pet.
2. manager → robot: **signed** `Switch {app: <id>, seq}`.
3. robot → manager: `State {question: "Connect to Raw data?", seconds_left: 60}`; the page:
   "Tap Yes on the robot (60 s)", the button waits.
4. The person taps Yes: robot → the app: connects; → manager: `State {app: raw, answer:
   switched}`; the page: "On Raw data ✓". A No or the timeout: `answer: not confirmed`, and
   the page says so.
5. home manager → sm.w42.eu over the link: the new state (shadow).

**S7. Switching on the robot's screen**
1. The person at the robot: QR screen → Next → Connect.
2. robot → manager: `State {app}`; the pages (primary, and the shadow over the link) follow.

**S8. Removing a paired browser**
1. Apps report their paired browsers to the manager that issued the robot's token
   (robot-auth `seen.pairings`): the home's apps to the home manager; raw.sa.w42.eu to
   sm.w42.eu, which passes them down the link.
2. page (primary): all browsers by app; **Remove**.
3. For a home app: home manager → that app: `unpair` in its next robot-auth answer. For an
   sm.w42.eu app: home manager → sm.w42.eu over the link: `Unpair {robot, app, ids}`; sm.w42.eu
   → raw.sa.w42.eu the same way.
4. A browser with end-to-end encryption: manager → robot: **signed** `Forget {browsers}`; the
   robot forgets it (a new group key) → `State {forgotten}`; the page: "out ✓".

**S9. A hung robot**
1. The robot's channel task gets no heartbeat from the app loop for 10 s → manager:
   `State {stuck: true}`.
2. page: **Restart the robot** is always on the card but greyed out while the robot answers;
   now it is highlighted, with "The robot is not responding".
3. manager → robot: **signed** `Restart {seq}`; the channel task restarts the robot; it comes
   back on its start app (S1, 5–6).

**S10. The internet is down**
1. The link to sm.w42.eu drops; sm.w42.eu shows "home on laptop not connected since 18:02" and
   the last known state.
2. At home nothing changes: robots, the home's apps and the home manager's page work.
   sm.w42.eu's apps (raw.sa.w42.eu) are out of reach, as any internet app.
3. The link comes back: `Robots` again; sm.w42.eu catches up.

**S11. The home manager is off**
1. The robot's channel retries with backoff; its apps keep working. They check tokens with
   their issuers: the home's apps with the home manager, which is off, so a robot they confirmed
   within the last hour still connects (robotauth's grace) and others wait until it is back.
2. Nothing changes the robot remotely; the person at the robot can still switch on its screen.

**S12. Another primary**
Only over USB: S1 or S2 again replaces the key, the channel URL and token, and the whole app
list. The old primary's tokens stop working when it removes the robot.

**S14. Finding the home manager**
1. On the robot: the Manager screen (a button on the QR screen) shows "home on laptop",
   connected or not, and its page's address (`http://192.168.1.10:8790`) with a QR code: scan it
   with a phone on the home Wi-Fi.
2. On sm.w42.eu (read-only shadow): **Manage at home** at the top of the home's robots. With the
   local address shared (S3), it opens that address (it works on the home network); without it,
   it says "Open the home manager on your home network: its address is on the robot's Manager
   screen".

**S15. Full control through the home**
1. page (sm.w42.eu), the home linked with "sm.w42.eu may change the robots": the owner opens Pet.
2. sm.w42.eu → home manager over the link: `Request {id, robot, Switch {app}}`.
3. home manager: checks the mode, signs `Switch` (S6, 2–5), logs "from sm.w42.eu" in the robot's
   history → sm.w42.eu: `Result {id, ok}`; the robot's states follow over the link as in S6.
4. The same for `Apps`, `Forget`, `Restart`. In read-only mode the home answers `Result {id,
   refused}` and sm.w42.eu shows no buttons that change the robot.

**S16. The other manager becomes the primary, on the robot**
1. The USB setup gave the robot a second manager (key, channel URL, token) with **may become
   primary on the robot** on.
2. The person at the robot: Manager screen → **Use sm.w42.eu** → "Make sm.w42.eu this robot's
   manager?" → Yes.
3. robot: drops the channel to the old primary, opens it to the new one, `Hello`; the new
   primary's app list applies from then on (the old primary's apps stay until it sends its
   own). The old primary sees "managed by sm.w42.eu now" when the robot is back in reach.
4. Without that setting the screen only shows the manager; changing it takes USB (S12).

**S17. The manager off (USB only)**
1. On the robot: Manager screen → **Turn off** → "Turn home on laptop off?" → Yes. Or on the
   manager's page: ⋯ → **Turn the manager off (USB only)** → the manager sends a signed `Disable`
   (when the robot is away: when it connects).
2. robot: saves the manager as off, tells it `Off {by: robot | manager}`, closes the channel and
   opens none. The page: "The manager is off on this robot …"; nothing that changes the robot
   (the API answers `manager_off`).
3. Apps then change only over USB; the person switches apps on the QR screen.
4. On again only at the robot (Manager screen → **Turn on** → Yes) or by a USB setup: the
   channel's `Hello` tells the manager. A manager can never turn itself on.

**S18. A phone at home**
1. The home manager (no sign-in) trusts only the computer it runs on. While the robot's channel
   is up, it sends the robot a one-time page code (`PageCode`, valid 10 minutes, used once).
2. The Manager screen's QR is `http://<home>:8790/phone?code=…` (the robot shows only an address
   on its manager's own page). A phone that scans it is signed in there ("this phone ·
   Sign out"); the robot gets a new code.
3. Being at the robot is the proof, as for pairing with an app.

**S19. Set up with another manager over USB**
1. page (another manager) → robot over USB: a setup with a manager whose key differs from the
   primary's.
2. robot → old primary, on its open channel: `Leaving {to: "<the new manager's name>"}`, then
   the robot saves the new manager and connects to it. (Not connected: nobody is told.)
3. old primary: the robot "left": its card says "Set up with ‹to› over USB … it talks only to
   that one now", changes are refused (`409 left`), history notes it. A home sends it so to
   sm.w42.eu too.
4. A USB setup with the old manager again clears it (its Hello on the new channel).

**S13. Firmware over the air (later)**
manager → robot: **signed** `Firmware {version, manifest URL, SHA-256s}`; the robot asks on its
screen, downloads, checks, installs, reports. Designed separately.

## 5. Messages

**The manager channel** (robot ↔ primary), JSON frames like the wire protocol:

| Direction | Kind | Body |
|---|---|---|
| robot → | `Hello` | firmware, the app list version, the app it is on, its apps as `{id, name}` (`id`: the first 8 bytes of SHA-256 of the URL, hex) |
| robot → | `State` | on every change: app, connection state, `question {text, seconds_left}` or none, the last `answer` (`switched`, `not confirmed`, `refused: …`), `stuck`, `forgotten` |
| robot → | `Ping` | every 25 s when nothing else went |
| robot → | `Off` | `{by: "robot" \| "manager"}`: the manager is off on the robot now (S17); the channel closes |
| robot → | `Leaving` | `{to}`: a USB setup gave it another manager (S19); the channel closes |
| → robot | `PageCode` | `{url}`: a one-time sign-in address on the manager's page for the Manager screen's QR (S18) |
| → robot | `Disable` | **signed**: `{robot, seq}`: the manager turns itself off for this robot (S17) |
| → robot | `Apps` | **signed**: `{robot, version, servers [{name, url, token?}], pin, ask_pin}` |
| → robot | `Switch` | **signed**: `{robot, seq, app}` |
| → robot | `Forget` | **signed**: `{robot, seq, browsers}` |
| → robot | `Restart` | **signed**: `{robot, seq}` |

`seq` grows per robot; the robot keeps the last one in NVS and refuses older ones (no replays).
`version` does the same for `Apps`.

**The link** (home manager ↔ sm.w42.eu):

| Direction | Kind | Body |
|---|---|---|
| home → | `Robots` | `[{id, name, state}]` after connecting |
| home → | `State` | `{robot, state}` on every change |
| home → | `Grant` / `Revoke` | `{robot, app}`: a token for one of sm.w42.eu's apps, or its end |
| sm → | `Granted` | `{robot, app, url, token}` |
| sm → | `Pairings` | `{robot, app, pairings}` as its apps report them |
| home → | `Unpair` | `{robot, app, ids}` |
| home → | `Home` | `{name, mode: "read-only" \| "full", local_url?}` after connecting and on a change |
| sm → | `Request` | `{id, robot, kind, body}`: a change from sm.w42.eu's page (full control only) |
| home → | `Result` | `{id, ok \| refused \| error}` |

## 6. Trust and privacy

- **The robot trusts only its primary's key** (principle 3: the home is the root of trust).
  sm.w42.eu never signs for a home's robot. As a read-only shadow, a compromised sm.w42.eu can
  show wrong states and refuse tokens for its own apps (denial of service), nothing more
  (principle 4). With full control the owner trusts it more, knowingly: it can ask for anything
  the page can do, and the home manager signs it; every such change is in the home's log as
  "from sm.w42.eu", and the home can turn it off at once.
- **The home's local address** goes to sm.w42.eu only if the owner shares it (S3); otherwise
  only the robot's Manager screen shows it.
- **The channel's connection itself is not trusted:** what changes the robot is signed. A plain
  `ws://` on the home LAN is fine for that; a listener there sees the robot's state.
- **What sm.w42.eu learns of a linked home:** its robots' names, which app each runs (by name),
  their state. Not the home's app URLs (apps go up as `{id, name}`), not tokens of home apps.
- **Later:** the shadow can become a blind relay to the home manager's own page
  (architecture §10): remote control with nothing readable on w42.eu.

## 7. On the robot

- One primary instead of several managers: NVS `embody/manager {key, name, url, token, seq,
  ask_pin}`, and optionally `embody/manager2` with `may_become_primary` (S16); every app on the
  robot comes from the primary. The "several managers" code goes (principle 27).
- The **Manager screen**: a button on the QR screen; the manager's name, the channel's state,
  the address of the home manager's page with a QR code (with a one-time sign-in code for a
  phone, S18), **Turn off / Turn on** (S17), **Use ‹the other›** when allowed.
- The channel task: its own WebSocket client (reused from the app client), reconnect with
  backoff, a heartbeat from the app loop, `Restart` handled in the task itself.
- Signed messages are applied in the app loop under the screen lock (as app lists are since
  embody-v0.4.1).
- Memory: one more connection, `ws://` at home; `wss://` (one TLS session, roughly 40–50 KB)
  only for robots with sm.w42.eu as their primary. To be measured.

## 8. Decided

- One primary per robot, chosen over USB; sm.w42.eu is the primary without a home manager.
- A linked home is a read-only shadow on sm.w42.eu by default, with **Manage at home** at the
  top; full control through the home is the owner's choice per home (S3, S15).
- Sharing the home manager's local address with sm.w42.eu is optional; the robot's Manager
  screen always shows it (S14).
- The person at the robot may make the second manager the primary, if the USB setup allowed it
  (S16).
- **Restart the robot** is always on the card, greyed out while the robot answers, highlighted
  when it is stuck (S9).
- An sm.w42.eu account may link several homes.
- The manager is optional (S17): off on the robot or by the manager; on again only on the
  robot or over USB.
- A phone signs in to a home manager with the one-time code on a robot's Manager screen (S18).

## 9. Steps

1. Firmware: one manager, the channel task (Hello, State, signed Apps, Switch, Forget,
   Restart with `seq`), the heartbeat; the USB setup carries the channel URL and token.
2. Manager: `/robot` (channel), live state, the page (S5–S9); `/link` on sm.w42.eu and the
   home side of the link (S3, S4, S8, S15), the shadow page with Manage at home (S14), several
   homes per account.
3. Firmware: the Manager screen, the second manager (S14, S16).
4. Raw and pet: drop the ManagedApps relay, AppsVersion and the polling; keep robot-auth and
   pairings.
5. Tests: a fake robot channel and a fake link in the manager's tests and page tests; on the
   robot: S1, S5–S9 with a Yes and a timeout, S4 with raw.sa.w42.eu, S10 by pulling the link.
6. Release together (firmware, manager, raw, pet); robots are set up again over USB.

## 10. Other options considered

- **Today: managers through the apps** (robot-auth polling, the app relays signed lists). No
  reach outside the manager's own apps, 15 s delays, blind to questions, gone when the app
  hangs.
- **A. A connection to every manager at once.** Live everywhere, but several keys and
  connections on the robot, and two authorities that can disagree.
- **B. One connection, the managers in turn.** Saves memory with several TLS managers; the
  others wait up to a cycle plus a handshake: the delay this design removes.
- **C. A connection to the manager of the active app only.** One connection; but only that
  manager can act, and the other page can only point to it.

## 11. How it was tested (2026-10-06)

The steps that need a person at the robot (a tap, a Yes) ran with a test build
(`./container.sh test`: USB control without a Yes, taps may answer the robot's questions; never
released), driven over USB (`s-w42-eu-usb swipe`, `tap`, `screenshot -top`).

Three ways: Go tests with a simulated robot (s-w42-eu-manager `internal/robotsim`: the firmware's
channel logic, signature and seq checks included) against real managers over real WebSockets;
headless Chrome page tests (`e2e/`); and the real robot (CoreS3, this firmware) with a local
topology: the home manager (:8790) linked to a second manager standing in for sm.w42.eu (:8792,
`-accept-links`), each with its own Raw data server.

| Sequence | Go and page tests | Real robot |
|---|---|---|
| S1 setup with the home manager | ✓ | ✓ USB setup, channel up, page live |
| S2 setup with sm.w42.eu only | ✓ (the same channel) | — |
| S3 link | ✓, and in the browser (approval page and back) | ✓ (local sm) |
| S4 an sm.w42.eu app for a home robot | ✓ (token from sm, robot-auth at sm) | ✓ the robot connects to sm's Raw data with sm's token |
| S5 change apps | ✓ | ✓ list applied within a second |
| S6 switch, question, answer | ✓ Yes and timeout | ✓ question on the page, Yes tapped: switched; nobody: "not confirmed" after 60 s; without asking: on the new app in 1.3 s (10 of 10, and more) |
| S7 switch on the robot's screen | ✓ | ✓ QR screen: Next, Connect; the page follows |
| S8 remove a browser | ✓ home app and sm app | ✓ both; the robot forgot the end-to-end browser (new epoch) |
| S9 a hung robot | ✓ | ✓ (`s-w42-eu-usb stall`): "not responding" after 11 s, Restart from the page, back in 10 s |
| S10 the internet (sm) down | ✓ | ✓ home kept working; link back 2 s after sm |
| S11 the home manager off | — | ✓ robot stays on its app; channel back 23 s after the manager |
| S12 another primary | ✓ | ✓ (set up again several times) |
| S14 finding the home manager | ✓ Manage at home (shared address) | ✓ the gear, the Manager screen with its page's address and QR |
| S15 full control through the home | ✓, and in the browser | ✓ switches from sm's page in 1.3 s; read-only answers 403 |
| S16 the second manager | ✓ there and back | ✓ there and back on the Manager screen: each manager's own apps follow |
| U17 several homes | ✓ | — |
| S17 the manager off | ✓ on the robot, from the page, while away | ✓ off and on on the Manager screen (Yes); off from the page; on by a USB setup |
| S18 a phone at home | ✓ | ✓ the Manager screen's QR signs a phone in; used once, then a new code |
| S19 set up with another manager | ✓ there and back, and in the browser | ✓ (2026-10-06, test build) home → the local sm over USB: home's page "Set up with sm test over USB" at once; back home: cleared, the robot under its home on sm again |

Found and fixed on the way: a PMIC read that aborted the firmware on an I2C timeout during the
clean start after an app switch (now skipped, and the clean start changes only what is not at
its default); a switch lost with a dropping channel (sent again when the robot is back within
30 s).

Catalog apps the robot already has a token for, from another manager (`"robot_token": true`),
keep a robot's sm.w42.eu apps while its home manager is its primary and the home is not linked
yet.
