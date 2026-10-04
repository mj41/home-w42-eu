# Use cases

**Status:** draft, 2026-10-04. **This is the main driver.** Every feature, protocol
and data flow in this project exists to serve one of these use cases. A stage on the
[roadmap](fit-and-roadmap.md) is done when its use case works for the people in it,
not when the code runs.

Each use case says **who**, **what they need**, **how it feels when it works**, what it
takes, what must stay private, and where it stands. The **For** column names the people by
their numbers in [Personas and agents](personas.md); the programs that serve each use case
(agents, loops, apps, the relay) are listed there too.

## Overview

| # | Use case | For | Status |
|---|---|---|---|
| 1 | [Be there from anywhere](#1-be-there-from-anywhere) | owner (1), family far away (5), another owner (7) | works on the LAN (Stackchan dashboard, sbot cockpit) and through raw.sa.w42.eu (sign-in, private robots) |
| 2 | [A friend for the kids](#2-a-friend-for-the-kids) | kids (3), another owner (7) | works (the pet) |
| 3 | [Play and explore together](#3-play-and-explore-together) | owner at play (2), kids (3) | works (Stackchan on the TPBot, joystick, safety stop, `frown`) |
| 4 | [Kids got home safely](#4-kids-got-home-safely) | owner (1), kids (3), other adults (4) | next: Wi-Fi presence, NFC card |
| 5 | [A comfortable home that saves energy](#5-a-comfortable-home-that-saves-energy) | owner (1), other adults (4) | next: Home Assistant data, window shutters controller |
| 6 | [The home looks after itself when we are away](#6-the-home-looks-after-itself-when-we-are-away) | owner (1), other adults (4) | idea |
| 7 | [Who is at the door](#7-who-is-at-the-door) | owner (1), other adults (4) | idea |
| 8 | [A home that knows our days](#8-a-home-that-knows-our-days) | owner (1), other adults (4) | idea (calendars) |
| 9 | [A guest or a babysitter for an evening](#9-a-guest-or-a-babysitter-for-an-evening) | owner (1), a guest or a babysitter (6) | partly (QR pairing) |
| 10 | [Build what we want by asking](#10-build-what-we-want-by-asking) | owner (1), owner at play (2), developer (8) | partly (hub with replay, controller server; the agent gateway is design) |
| 11 | [Our data stays ours](#11-our-data-stays-ours) | everyone, the owner (1) first | partly (local servers, media only while watched) |
| 12 | [A second life for old devices](#12-a-second-life-for-old-devices) | owner at play (2), developer (8) | idea |
| 13 | [Follow up on what I saved](#13-follow-up-on-what-i-saved) | owner (1) | idea |
| 14 | [Approve what matters, where it is safe](#14-approve-what-matters-where-it-is-safe) | owner (1), independent rebuilder (9) | partly (release approval) |
| 15 | [Set up my robot with one click, privately, and check what I install](#15-set-up-my-robot-with-one-click-privately-and-check-what-i-install) | another owner (7), independent rebuilder (9) | works (sm.w42.eu/setup; releases checked in mj41cz-approved) |

---

## 1. Be there from anywhere

- **Who:** a parent at work or travelling.
- **Need:** see and hear the home, talk to the kids, look around, without a vendor app
  or a cloud that watches too.
- **When it works:** I open one page, see through Stackchan's eyes, turn its head
  towards the kitchen, say hello through its speaker, and it frowns or smiles as I
  choose. At home it is instant; away it still works, and nobody else could watch.
- **Takes:** Stackchan (camera, mic, speaker, head), the cockpit, local-first
  connection, the hub when away.
- **Private:** media only while watched, LIVE badge on the robot, no recording by
  default; robots in kids' rooms LAN only.
- **Status:** works on the LAN, and today through raw.sa.w42.eu with sign-in, private robots and
  end-to-end encryption available. Away from home without a relay that could read: the blind
  hub (roadmap stage 10).

## 2. A friend for the kids

- **Who:** kids at home.
- **Need:** a companion that is fun, gentle, and part of their day: hungry in the
  morning, sleepy at night, happy when they come back from school.
- **When it works:** the kid feeds the pet with an NFC food card, plays the colour
  game, and the pet naps when it is bedtime. Parents set the rules behind a PIN.
  Later: Ema taps her own card on any robot in the house and her pet appears there;
  ten minutes after she stops playing, the robot goes back to its normal face.
- **Takes:** Stackchan, the pet app, NFC cards; later the calendar (use case 8).
- **Private:** the kid's data is the kid's (and the parents'); game photos stay on the
  node.
- **Status:** works.

## 3. Play and explore together

- **Who:** kids and parents.
- **Need:** a robot that can move around the home, driven like a game, safely.
- **When it works:** we put Stackchan on the car, drag the joystick on a phone, and
  look through its eyes as it drives down the corridor. It stops by itself before
  the wall, and the face shows it.
- **Takes:** the TPBot car with the micro:bit, Stackchan hosting it over BLE, the
  cockpit, the safety stop, the robot reacting to the car (`frown`).
- **Private:** the camera is on only while someone drives.
- **Status:** works (POC): a car command takes about 0.2–0.3 s through Stackchan; the
  micro:bit accepts only listed BLE centrals (an allowlist); the robot frowns while the car is
  blocked (`frown`, live).

## 4. Kids got home safely

- **Who:** parents at work.
- **Need:** to know that a kid came home from school, without tracking the kid all
  day.
- **When it works:** at 13:40 my phone says "Ema is home". The pet greets her by name.
  If she is not home by 15:00 on a school day, I get a gentle note, not an alarm.
- **Takes:** Wi-Fi presence (her phone joined), or her NFC card on the robot; the home
  model (who Ema is, what a school day is); a loop; the calendar later. Her own
  permissions follow her card: on weekday afternoons she may drive the car.
- **Private:** only "home / not home", never her location outside; she can see what
  was recorded and who read it.
- **Status:** next (roadmap stages 3–7).

## 5. A comfortable home that saves energy

- **Who:** everyone living there.
- **Need:** rooms that are not too hot in summer or cold in winter, without anyone
  running around closing shutters, and lower energy bills.
- **When it works:** on a hot afternoon the shutters on the sunny side close before the
  rooms heat up, and open again when the sun moves. In strong wind they go up to stay
  safe. Pressing the wall switch always wins.
- **Takes:** Home Assistant as the data source (temperatures, sun, weather, wind), the
  existing custom shutter hardware and software, a controller on the node.
- **Private:** nothing leaves the home; presence data only as "someone is home".
- **Status:** next (roadmap stages 3–4).

## 6. The home looks after itself when we are away

- **Who:** the family on a trip, or out for the day.
- **Need:** the house cleaned, the heating low, and a heads-up if something is wrong.
- **When it works:** when the last phone leaves, the vacuum starts and the heating goes
  down. If a window opens while nobody is home, I get a message with a snapshot.
- **Takes:** Wi-Fi presence, Home Assistant (contacts, heating), a vacuum adapter,
  cameras, loops.
- **Private:** snapshots only on that event, short retention.
- **Status:** idea.

## 7. Who is at the door

- **Who:** whoever is home (busy in another room), or away.
- **Need:** to know who is at the door, or that a parcel was left.
- **When it works:** Stackchan turns to me and says "a parcel at the door"; the cockpit
  shows the door camera. A family car arriving opens the gate.
- **Takes:** a door camera (an old phone works), person and car detection on the node,
  known plates only, loops.
- **Private:** cameras point at our own property only; unknown faces and plates are not
  named or kept.
- **Status:** idea.

## 8. A home that knows our days

- **Who:** the family.
- **Need:** routines that follow real life: school days, holidays, late meetings.
- **When it works:** on a school holiday the pet does not wake the kids at seven; the
  heating knows we are away until Sunday.
- **Takes:** calendar adapters, the home model (routines), loops.
- **Private:** calendar titles stay with their owner; the home sees only "busy",
  "away", "school day".
- **Status:** idea.

## 9. A guest or a babysitter for an evening

- **Who:** parents, and a babysitter or a visiting friend.
- **Need:** give someone just enough of the home for a few hours.
- **When it works:** the babysitter scans the QR on the robot and can see the hall
  camera and switch the lights until midnight, but not the parents' rooms or
  history.
- **Takes:** permission entries per person, device, capability and time ("hall camera
  and lights, for 5 hours"), QR and NFC tokens.
- **Private:** the guest's access is audited and ends by itself.
- **Status:** partly (QR pairing works; guest scopes are design).

## 10. Build what we want by asking

- **Who:** the owner, who knows what they want but not how to program it.
- **Need:** new behaviour without writing a controller by hand, and without giving an
  AI control of the home.
- **When it works:** I say "when the car is blocked, the robot should look sad". The
  assistant writes it, shows me what it would have done last week, I approve it, and
  it runs. I can switch it off any time.
- **Takes:** the event hub with replay, the controller server, the agent gateway,
  owner approval.
- **Private:** the AI sees only the events its task needs, and nothing sensitive by
  default.
- **Status:** partly: the event hub with replay and the controller server work, and the
  first loop between two devices, `frown`, went replay → shadow → live. The agent gateway is
  design.

## 11. Our data stays ours

- **Who:** everyone in the home.
- **Need:** to know what is kept about me, who looked at it, and to delete it.
- **When it works:** I open "my data" and read in plain words: "the cockpit read 3 hours
  of the car's sensors; the presence loop checked if you were home 12 times". I tap
  delete and it is gone, with its key.
- **Takes:** the event hub with retention, read audit, separate stores, owner keys.
- **Private:** this is the privacy use case.
- **Status:** partly: data stays on our own servers, media streams only while watched;
  the "my data" page, audit and keys are design.

## 12. A second life for old devices

- **Who:** the family (and the planet).
- **Need:** use the phones, tablets and laptops in the drawer instead of buying new
  "smart" devices.
- **When it works:** the old phone is the door camera, the old tablet is the cockpit on
  the kitchen wall, the old laptop is the home node.
- **Takes:** light clients for phones and browsers, adapters.
- **Private:** old devices on a separate network that reaches only the node.
- **Status:** idea.

## 13. Follow up on what I saved

- **Who:** an adult who saves things in many places and forgets them.
- **Need:** one place that collects "for later" items (a bookmarks folder, saved posts
  on X and LinkedIn, links and promises in a family Messenger group, task lists, the
  calendar) and brings them back at the right time, without handing an account to
  anyone.
- **When it works:** on Sunday evening the home shows me: "12 things saved this week:
  4 articles to read, 2 replies you promised in the family group, 1 task due Tuesday".
  I mark what is done; the rest comes back next week. My Google or Facebook account
  was never opened by anything but me.
- **Takes:** a capture station on my laptop with tiny reviewed collectors (one folder,
  one group, one list), "share to home" from my phone, captures in my own store, a
  digest loop with a local AI model.
- **Private:** the captures are mine only; no credentials leave the laptop; sources
  that would need full account access are not collected.
- **Status:** idea.

## 14. Approve what matters, where it is safe

- **Who:** an owner or a parent.
- **Need:** devices and apps may need something that takes a secret (a payment, a
  signed release, a login), and the secret must never sit on a robot, a kitchen tablet
  or a server.
- **When it works:** the kids ask the robot to order pizza. My phone buzzes: "Pay 18.40
  EUR to Pizza Roma for the kids' dinner?" I check it and approve with my fingerprint;
  the robot hears "ordered, arriving at 19:10" and never saw my card. On another day an
  agent has a new sbot release ready; my laptop shows the version and its hash, I touch
  the YubiKey, and the release is signed and published.
- **Takes:** delegated actions (architecture §8.7), holder devices in their perimeters,
  permission entries for who may ask for what, and up to how much.
- **Private:** card data stays on my phone; signing keys stay on my laptop or key; the
  requester only gets the result.
- **Status:** partly: firmware releases are approved on the owner's laptop: its
  `SHA256SUMS` signed there and published in
  [mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved)
  ([device setup](device-setup.md) §6.2). Delegated actions are design; the owner-key signing
  of the trust design is the first such perimeter.

## 15. Set up my robot with one click, privately, and check what I install

- **Who:** someone with a new robot who uses sm.w42.eu or their own server, and anyone who
  wants to check the firmware before trusting it ([device setup](device-setup.md) §1).
- **Need:** a robot that works without a terminal, private to them, with firmware they can
  check is what its source says.
- **When it works:** I plug the robot in, open sm.w42.eu/setup in Chrome, pick the apps and press
  one button. The page backs up the robot's firmware, installs Embody Mode and connects the robot;
  I tap Yes on the robot's screen, and the page opens my robot, paired. Nobody else can use
  it. If I want, I compare the firmware with the hashes signed in mj41cz-approved, or rebuild
  it from its tag and get the same bytes.
- **Takes:** the setup page and the USB setup protocol with the tap on the robot, sign-in and
  private robots, reproducible releases with signed hashes ([device setup](device-setup.md)
  §6); later the flasher as its own page.
- **Private:** no server, token or Wi-Fi inside the firmware; robots private to their owner by
  default; end-to-end encryption available.
- **Status:** works at sm.w42.eu/setup and with a manager at home; releases are rebuilt and
  approved in mj41cz-approved. Planned: the flasher as its own page, and the owner's approval
  before a release is published.

---

## Adding a use case

Start from a person and a moment in their day, not from a device or a protocol. Write
**who**, **need**, **when it works** (as they would tell it), **takes**, **private**,
**status**. Then check which devices, data and loops it needs; if something is
missing, that is the next thing to build.
