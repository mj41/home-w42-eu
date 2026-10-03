# Vision

**Status:** draft, 2026-10-02.

> **Your home runs on your own small server. Every device in it, new or old, is a
> plain client of that server. Nothing leaves the home unless you sign that it may.
> AI helps you build the loops that make the devices work together, and you approve
> every one of them.**

## Use cases first

This project is driven by **real use cases for people**, not by data or protocols:
a parent who wants to know the kids got home, kids who want a friend that knows
their day, a home that stays cool without anyone running around closing shutters.
Every part of the platform exists because a [use case](use-cases.md) needs it.
Raw data and protocols are the plumbing; **meaning** (who, where, what it is for)
is made on the home node, in the words the family uses.

## The problem

"Smart" devices today are not smart, and they are not connected:

- **Each one talks to its own vendor cloud.** A robot vacuum, a doorbell, a lawn
  mower and a thermostat from four brands are four apps, four accounts and four
  companies that see into your home. They cannot talk to each other except through
  those clouds, if at all.
- **The data is not yours.** Location history, camera footage, presence and voice
  are stored where you cannot see, export or delete them.
- **The devices die with the vendor.** When the cloud closes or the app stops
  being updated, working hardware becomes waste. A phone from five years ago has a
  camera, microphone, GPS, screen and battery, and it sits in a drawer.
- **Automation is either too simple or too hard.** "If this, then that" rules
  cannot express real behaviour; writing a proper controller needs a developer.

## What we build instead

A **local first, security and privacy first platform** for one home (or family),
written in **Go**, built from these parts:

1. **A home node.** A small machine at home (an old laptop, a Raspberry Pi, a
   mini PC) running Go servers: a web/API server for devices, browsers and apps, and
   a controller server for loops, both around the home's event hub. It holds the
   home's data and keys, and works with no internet at all.
2. **Light clients.** Every device is a thin client: it connects out to the node,
   says what it can do (commands, measurements, events), sends its **raw data**,
   and does what it is told. The intelligence lives on the node, not in the
   firmware. A Stackchan robot, a micro:bit car, an old Android phone, an ESP32
   sensor and a browser tab are all the same kind of thing.
3. **Adapters for what cannot speak for itself.** Old and closed hardware joins
   through a small adapter: BLE (the TPBot car today), serial (an old Roomba),
   a router API (who is on the Wi-Fi), Home Assistant (its whole sensor array).
4. **Apps you can switch between.** A device is not married to one function. The
   same Stackchan is a telepresence robot, a pet for the kids, or the head of a
   small car, depending on which app it is connected to. Switching is a tap on the
   device or a click in the browser.
5. **One event hub.** Everything that happens is an event in the home's
   append-only, log-based hub (like Kafka, sized for a home): raw sensor readings, button presses, commands, who sent them,
   and what each loop decided. It is the home's memory, its audit trail, and the
   ground truth that loops are tested against.
6. **Loops and controllers.** Behaviour is code that reads events and sends
   commands: "stop the car before the wall", "when the last phone leaves the
   Wi-Fi, start the vacuum", "when the doorbell rings, turn the robot's head to
   the door and show the camera". Each loop is small, sandboxed and allowed only
   the capabilities it was granted.
7. **AI as the builder, not the operator.** You describe what you want in plain
   words. An AI agent writes the loop, tests it against your recorded events,
   shows you what it would have done, and asks for approval. The agent never holds
   your keys and never acts outside the scopes you sign. The harness that runs
   agents is a client of the node like any device.
8. **A hub only for when you are away.** At home, every device and browser talks
   to the home node directly, with no detour. From outside, you reach the home node
   through `w42.eu`, which only forwards encrypted bytes it cannot read. It is a
   hub for finding the home, not a cloud, and some devices never use it at all:
   a camera in a kid's room can be LAN only, so its data cannot leave the house.

## What it should feel like

- You plug in an old phone, scan a QR code on the node's page, and the phone's
  camera, GPS and screen are now part of the home, visible only to you.
- The robot on your desk notices that the car's sonar sees an obstacle and
  frowns, because you once said "the robot should react when the car is stuck".
- You ask "what did the cameras see while we were away?", and the answer comes
  from your own node, from data that never left the house.
- A device you no longer trust is revoked with one signature, and its history is
  deleted with its key.
- Nothing breaks when the internet is down, and nothing breaks when a company
  closes.

## Who it is for

- **First, us:** one family, real hardware, real use. Every part is proved on our
  own home before it is described as done.
- **Then, tinkerers and families** who want this without writing the platform:
  open source, reproducible builds, documented hardware, and AI help for the parts
  that used to need a developer.
- **Not:** a product that needs an account with us. The w42 services are
  conveniences (relay, rendezvous, backups of encrypted bytes). A home that never
  uses them loses nothing but remote access.

## What we do not do

- No vendor clouds in the data path. Where a device requires one, it gets an
  adapter or it stays out.
- No derived data in device firmware. Devices report what they measure; meaning
  is made in loops, where it can be seen, tested and changed.
- No AI with standing authority. Agents propose; owners sign.
- No data kept "just in case". Retention is short by default, and longer only by
  an owner's choice (see [principles](principles.md)).

## How we get there

Use case by use case: each stage makes one of the [use cases](use-cases.md) work for
real people, proved on real hardware, and leaves something that works.
The current state and the next stages are in [fit-and-roadmap](fit-and-roadmap.md);
the design is in [architecture](architecture.md); the first full device family is
[Stackchan](implementations/stackchan.md); new device and app ideas live in
[home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas).
