# Principles

**Status:** draft, 2026-10-02. **Binding** on every repo that is part of home-w42-eu.
Where a repo's own rule is stricter, the stricter rule wins.

They collect the data and trust rules designed earlier for w42 and for Stackchan
([design.md](https://github.com/mj41/stackchan-mj/blob/main/docs/design.md) in stackchan-mj), and what the work on Stackchan, sbot and the
TPBot car taught.

## Value

1. **Real use cases drive everything.** Every feature starts from a person's need in a
   moment of their day, written as a [use case](use-cases.md). Data, protocols and
   hardware serve it. A feature is done when its use case works for the people in it.
2. **Meaning lives on the node, in words people use.** Devices report raw data; the
   node's home model (people, rooms, places, things, routines) gives it meaning, and
   loops turn readings into what people care about: "Ema is home", "the car is
   blocked", "a parcel at the door". Pages, notifications and the audit speak in
   those terms, not in MAC addresses and sensor names.

## Ownership and trust

3. **The owner is the root of trust.** One owner key per home (ECDSA P-256; on a
   laptop, a YubiKey, later a passkey) signs device certificates, grants,
   configuration and firmware manifests. Devices verify these signatures themselves.
4. **No service we run is trusted for authenticity.** A compromised `w42.eu` may
   deny service. It must not be able to redirect a device, impersonate one, or
   read a home's data.
5. **Devices enforce their grants.** Every command arrives tagged with the app (and
   user) it came from; the device drops what that app was not granted. The node
   filters too, as defence in depth.
6. **Restricted by default.** Nothing is allowed without a grant. Effective access
   is the intersection of the person, the device acting as their proxy, the app or
   loop, and the target device's own access list.
   Permissions are given **per person, per device, per capability and for a time**
   ("Ema may drive the car on weekday afternoons", "the babysitter may watch the hall
   camera for 5 hours"). Only owners and owner-approved loops hold permissions without
   an end.
7. **Trust has limits.** Every trust level (owner, family, kid, guest, app, loop,
   agent) has limits on how much it may read and how often it may act. Bulk reads
   and exports are rate limited, budgeted and logged; a caller reading far more than
   usual is paused.
8. **Media is never implied.** Camera, microphone, speaker and location are separate
   scopes, granted explicitly, and visible on the device while in use (Stackchan's
   red LIVE badge is the pattern). Sensitive data (location, calendars, presence)
   goes only to AI models the owner allowed for it, by default local ones.

9. **Third-party accounts only through narrow, reviewed collectors.** Captures from
   accounts (messages, bookmarks, saved posts, tasks) are collected only in a secure
   capture station on the person's laptop, by tiny reviewed collectors with the
   narrowest access to one part of the account. Credentials never reach the node or
   any AI agent.

10. **Secrets stay in their perimeter; others request actions.** Keys, account tokens
    and payment cards live in a defined perimeter (the owner's signing station, the
    capture station, a person's phone) and never leave it. A device that needs such an
    action sends a request; a device inside the perimeter shows it in plain words, the
    person approves it there, that device performs it, and only the result goes back.

## Data

11. **Local first.** The home works with no internet. Data and keys live on the home
   node and the owners' devices.
   On the home network, devices and browsers talk to the node directly; `w42.eu` is
   only a hub for finding the home and reaching it from outside, and never in a path
   that can be local. A device may be **LAN only** (relay policy `never`): its data
   leaves the house by no path.
12. **Primary data only from devices.** Devices report raw readings, hardware events
   and the state of their controls. Anything derived ("head moved by hand",
   "car is stuck", "someone is home") is made by a loop on the node.
13. **Every record has one home.** Per person, per household, per device, per app:
   separate stores, joined only by opaque ids. Deleting or exporting one is one file
   and one key.
14. **Short retention by default.** Hot data for 7 days; older data is sealed to the
   owner's public key, offered for download, then deleted. Longer retention is an
   owner's explicit choice, per kind of data.
15. **Encrypted, with rotating keys.** At rest on the node, and end to end through
   any relay. Deleting a key deletes the data it protected (crypto-shredding).

## Behaviour

16. **Everything is an event, and every action and read is logged.** Commands, who
    sent them, what each loop decided and why, and who read which data and how much.
    The log is the audit trail the owner can read in plain words.
17. **Loops are small, sandboxed and scoped.** A loop gets only the capabilities in
    its grant. A loop that misbehaves is stopped without stopping the home.
18. **AI proposes, owners approve.** Agents may read the events they were granted,
    write loops and test them against recorded events. Running a loop, or giving it
    more scopes, needs the owner's signature. Agents never hold owner keys.
19. **Safety lives as close to the hardware as possible.** A watchdog on the device
    (the micro:bit stops its motors 500 ms after the last command), a controller on
    the node (the sonar safety stop), and the UI last. No layer relies on the one
    above it.

## Devices and apps

20. **Devices are light clients.** They connect out, register their capabilities,
    stream raw data while someone wants it, and run commands. App logic stays on the
    node, so a device gains functions without new firmware.
21. **Extensions are optional.** A feature that needs extra hardware (the TPBot car)
    is compiled in but off until enabled, costs nothing while off, and never changes
    what other users see.
22. **Switching apps is simple, and it comes back.** A device keeps a list of servers and
    apps and can switch with a tap or a command. A scan (NFC, QR), a person arriving or
    a loop can switch a device's UI for a session, with that person's rights; when the
    session ends, the device returns to its default UI. The owner decides which apps a
    device may join.
23. **Old hardware is welcome.** If it has a radio, a port or a screen, it gets an
    adapter or a light client, before it gets thrown away.

## Engineering

24. **Go for servers, tools and bridges.** One static binary per role. Firmware in
    whatever the hardware needs (ESP-IDF C++, TinyGo for the micro:bit).
25. **Open source and reproducible.** Anyone can rebuild a firmware image or a node
    binary, compare the hash, and sign it with their own key.
26. **Proved on real hardware.** A feature is "done" when it worked on our own
    devices, with what was tested and what was not written down.
27. **Early stage: no backward compatibility.** Until a first stable release, protocols,
    APIs, file formats and stored settings change when something better comes along,
    without migrations or old names kept alongside: devices and servers are updated
    together. Changes are still written down (the wire protocol's change list), so a
    rebuild of an older release can be explained.
