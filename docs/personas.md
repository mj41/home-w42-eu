# Personas and agents

**Status:** draft, 2026-10-04. Who the [use cases](use-cases.md) are for: the people, and the
programs that act in a home. Every feature serves one of them, and every permission is decided
per persona: what it may do, what it must never do, and what matters most to it.

The home of the first deployment is mj41's; the personas are written so that any home fits.
Nothing personal about real people goes here (no names, ages or habits of the kids).

## Overview

| # | Persona | Kind | First of all | Use cases |
|---|---|---|---|---|
| 1 | [The owner (mj41)](#1-the-owner-mj41) | person | privacy and security | all, 10, 11, 14 |
| 2 | [The owner at play](#2-the-owner-at-play) | person | fun, no ceremony | 3, 10, 12 |
| 3 | [The kids](#3-the-kids) | people | fun, safety | 2, 3, 4 |
| 4 | [Other adults at home](#4-other-adults-at-home) | people | it just works | 4, 5, 6, 8 |
| 5 | [Family far away](#5-family-far-away) | people | be there | 1 |
| 6 | [A guest or a babysitter](#6-a-guest-or-a-babysitter) | people | easy, then gone | 9 |
| 7 | [Another owner on chan.w42.eu](#7-another-owner-on-chanw42eu) | people | one click, private | 1, 2 |
| 8 | [A developer](#8-a-developer) | people | clear, reproducible | 10, 12 |
| 9 | [An independent rebuilder](#9-an-independent-rebuilder) | people | verifiable | 11 |
| 10 | [Viewers of a public stream](#10-viewers-of-a-public-stream) | people | watch, nothing more | — (planned) |
| 11 | [The owner's AI agents](#11-the-owners-ai-agents) | program | do the task, nothing else | 10, 13, 14 |
| 12 | [Home automation (loops)](#12-home-automation-loops) | program | reliable, local | 4, 5, 6 |
| 13 | [The fun apps](#13-the-fun-apps) | program | fun first | 2, 3 |
| 14 | [The relay (w42.eu)](#14-the-relay-w42eu) | service | carry, not read | 1 |
| 15 | [The adversary](#15-the-adversary) | threat | — | all |

## People

### 1. The owner (mj41)

- **Who:** runs the home's servers, owns the devices, signs releases and approvals; a developer.
- **Wants:** to see and control everything at home from anywhere, without any company in the
  middle being able to watch, listen or drive.
- **First of all:** privacy and security. The owner key is the root of trust (principle 3); data
  stays at home unless the owner signs that it may leave; every release checkable
  ([device setup](device-setup.md) §6).
- **May:** everything, on their own devices; grant others and agents narrow, timed scopes.
- **Must never have to:** trust a service they do not run (w42.eu, GitHub, a cloud provider) with
  the content of the home.

### 2. The owner at play

- **Who:** the same person off duty: hacking on the robot, the car, a new app, with the kids.
- **Wants:** quick experiments: flash a build, try an idea, no tickets, no signatures in the way.
- **First of all:** fun, short loops from idea to robot.
- **The tension with persona 1** is resolved by **modes**, not by weaker rules: a development
  robot or a development server (own firmware, `stackchan-usb`, a LAN server without sign-in),
  separate from the home's production devices and releases. What is fun to build becomes a
  release through the normal, checked path.

### 3. The kids

- **Who:** children of the home; can read a little or not at all.
- **Wants:** a friend (the pet), a game, a car to drive, a robot that reacts to them.
- **First of all:** fun, at once, without accounts, passwords or setup; and **safety**: nobody
  outside the home sees or talks to them through a device unless a parent allowed that person;
  no ads, no tracking, nothing sent to a company.
- **May:** play with the fun apps (13) and their own devices; NFC cards and taps rather than
  logins.
- **May not:** change settings, pair strangers, start a stream to the outside, install apps.
- **Grows with them:** scopes widen with age (a teenager may get a login and an app of their own).

### 4. Other adults at home

- **Who:** a partner or another adult living there; not interested in how it works.
- **Wants:** it works without explanations; the home is comfortable; the kids got home.
- **First of all:** simple and dependable; a physical override always works (a touch, the power
  button, a switch).
- **May:** most of what the owner may at home; approvals for their own things.

### 5. Family far away

- **Who:** grandparents or a parent travelling.
- **Wants:** to say hello through the robot, see the kids, read them a story.
- **First of all:** being there, easily (a link, a QR code once), only when the family at home
  agreed; and the camera never on without the robot showing it (the LIVE badge).
- **May:** the scopes a grant gives (e.g. camera and speaker of one robot, evenings), through the
  relay with end-to-end encryption.

### 6. A guest or a babysitter

- **Who:** someone at home for an evening.
- **Wants:** to do what they are there for (put the kids to bed, play a game) without learning
  the system.
- **First of all:** easy now, gone afterwards: a grant until the morning, no lasting access, no
  history of the home.

### 7. Another owner on chan.w42.eu

- **Who:** someone with their own Stackchan who uses the public server instead of running one.
- **Wants:** plug the robot in, press one button, use the apps.
- **First of all:** one click ([device setup](device-setup.md) persona "anyone"), and their robot
  **private** to them (sign-in), with end-to-end encryption so the relay cannot watch.
- **May:** their own robots on the public server; the public apps.
- **Must not:** see or bother other people's robots; be bothered by anonymous browsers.

### 8. A developer

- **Who:** someone building an app, a device adapter or firmware for the platform.
- **Wants:** clear protocols, a local server in one command, a fake robot, the firmware built
  from source.
- **First of all:** clear and reproducible; early-stage changes written down (principle 27).

### 9. An independent rebuilder

- **Who:** someone who rebuilds a release on their own machine and signs its hashes
  ([device setup](device-setup.md) §6.2), or a cloud machine doing it for the owner
  ([independent rebuild](independent-rebuild.md)).
- **Wants:** one command, pinned inputs, the same bytes.

### 10. Viewers of a public stream

- **Who:** people on the internet watching a robot the owner chose to share live (e.g. on X or
  YouTube), anonymous or signed in.
- **Wants:** to watch, maybe to send a reaction the owner allowed.
- **May:** only what the owner shared, rate-limited; never anything that reaches the rest of the
  home.
- **Status:** planned (access rights for clients, tiers and rate limits are open notes).

## Programs and services

### 11. The owner's AI agents

- **What:** AI agents working for the owner: coding agents building the projects, and agents that
  act in the home (drive a robot, write a loop, sort what the owner saved).
- **Wants:** to finish the task they were given.
- **First of all:** do the task and nothing else: a grant with the narrowest scopes and an end
  time; every action in the audit log, tagged with the agent; anything risky or outward-facing
  (spending money, pushing, sending messages, unlocking, starting a stream) only after the owner
  approves it, where it is safe to approve (use case 14).
- **May not:** read what its task does not need; keep data; act as the owner; approve itself.
- **Automation on the robot** (Embody Mode's `automation`, `restart`, `launch`) is off by default
  and only for a server the owner trusts.

### 12. Home automation (loops)

- **What:** controllers and loops that run on their own: wake the robot when someone comes in, stop
  the car before a wall, shutters, energy.
- **First of all:** reliable and local: they work without the internet and without the owner's
  laptop; written down, testable on recorded events, switched off with one tap.
- **May:** only the scopes their grant lists (principle 5: devices enforce grants).
- **Never:** override a person at the device: a touch on the robot or its power button wins.

### 13. The fun apps

- **What:** the pet, the car cockpit, games: apps made for the kids and for play.
- **First of all:** fun: instant reactions, no friction, things that surprise; built for the kids
  (3) and the owner at play (2).
- **But:** the same rules as every app: their own data folder on the device
  ([device storage](device-storage.md)), no data out of the home, no camera or microphone unless
  the app needs it and the robot shows it.

### 14. The relay (w42.eu)

- **What:** the public server that connects robots and browsers that cannot reach each other
  (chan.w42.eu, auth.w42.eu).
- **Useful, not trusted** (principle 4): it may carry and drop, never read, redirect or drive
  ([e2ee](e2ee.md), [device setup](device-setup.md) §2).

### 15. The adversary

For the threat model, not a user: a stranger on the internet, a compromised relay or build
machine, a malicious app or server, a curious visitor with physical access to a robot. Each design
says what it can and cannot do against them (e.g. [e2ee](e2ee.md) §1,
[device setup](device-setup.md) §2).

## Priorities

What matters first, when two goals conflict:

| Persona | Privacy | Security | Fun | Simplicity | Control |
|---|---|---|---|---|---|
| The owner | 1 | 1 | 3 | 3 | 1 |
| The owner at play | 2 | 2 | 1 | 2 | 1 |
| The kids | 1 (decided for them) | 1 (decided for them) | 1 | 1 | — |
| Other adults | 1 | 2 | 3 | 1 | 2 |
| Another owner on chan.w42.eu | 1 | 2 | 2 | 1 | 2 |
| AI agents, loops | follow their grant | follow their grant | — | — | none of their own |

1 first, 3 later. For the kids, privacy and security are decided by the parents, never traded for
fun.

## Adding a persona

Who, what they want, what matters first, what they may and may not do, which use cases; then add
them to the use cases' **For** column and check the permissions a design gives them.
