# sm.w42.eu: one screen for your robot

**Status:** 2026-10-05, agreed; being built. Replaces the two pages "Your robots" and "Set a robot
up" ([stackchan-sites.md](stackchan-sites.md)).

## 1. Who and what

- **The usual person has one robot.** They set it up once over USB, then use its apps and
  change them now and then from any device, without the cable.
- **Some want no remote control at all.** Then everything that changes the robot goes over
  USB, and the page says so instead of offering buttons that cannot work.
- **A few have several robots** (a family, a classroom). The same screen lists them.

So the page is **your robot**, not a list of tools. Everything you can do with a robot is on
its card, and the page offers only what can work right now: online or over the cable.

## 2. The one screen

`sm.w42.eu` is the only page (`/setup` goes away). What it shows depends on the state:

| State | What the page shows |
|---|---|
| signed out | what this is, **Sign in** (GitHub, Google) |
| no robot yet | **Set up your robot**: plug it in, one button (§3) |
| one robot | its card, big (§4) |
| several | a card each, compact, the one used last open; **Add a robot** at the end |
| a robot on the USB cable | a USB strip on its card, or "a new robot is plugged in" (§5) |

```
┌──────────────────────────────────────────────────────┐
│ sm.w42.eu                          Ema · Sign out    │
├──────────────────────────────────────────────────────┤
│ ● Stackchan 0a1b…4e50              online · on Pet   │
│                                                      │
│ [ Open Pet ]                                     ⋯   │
│                                                      │
│ Apps   ★ Pet   Raw data   + add                 │
│        changes go to the robot online ✓              │
│                                                      │
│ Private · firmware embody-v0.3.0 (latest)            │
└──────────────────────────────────────────────────────┘
  + Add a robot
```

## 3. First setup (no robot yet, or "Add a robot")

One card, everything prefilled with the usual choice; the details are folded:

1. "Plug your Stackchan into this computer (the USB-C port on its head)." **Connect** opens
   Chrome's device chooser (Web Serial needs one click the first time).
2. The page asks the robot who it is (`hello`): its id and firmware. It says what it found:
   "A new robot, with its original firmware" or "embody-v0.2.0, set up before by you".
3. The choices, prefilled:
   - **Apps:** every app of the catalog ticked, the first one starred (it starts with it).
   - **Wi-Fi:** this computer's network if the browser can tell, else empty ("the robot opens
     a hotspot").
   - **Let sm.w42.eu change the apps later** ✓ and **Ask on the robot before the start app
     changes** ✓, in one line with a "Why?" note: the robot keeps the manager's key and
     accepts only lists signed with it.
   - Folded: back up the firmware first ✓, start Embody Mode when the robot turns on.
4. **Set up** → steps with progress → the card of the robot (§4), with **Open Pet**.

## 4. The robot card

- **Status** (from what the apps tell the manager when the robot connects): online or last
  seen, and on which app.
- **Open ‹start app›**: the big button. The start app is the one with the ★.
- **Apps:** chips; tap a chip to star it (start app) or remove it, **+ add** for the others.
  - Remote changes allowed: saved at once, with the state under it: "waiting for the robot"
    → "on the robot ✓" (when the robot reports the list's version), or "confirm on the robot's
    screen" for a new start app when it asks.
  - Remote changes not allowed: the chips are read-only, and the line says "Changes go over
    USB: plug the robot in" (the USB strip then offers it, §5).
- **⋯ menu** (rare things): make public/private, firmware (update over USB), permissions
  (change over USB), history, remove.
- **History** (folded): what was done to the robot, when and how: "set up over USB,
  embody-v0.2.0, Pet + Raw, remote changes allowed", "apps changed online: + Sbot", "firmware
  updated over USB". The manager keeps it; nothing secret in it.

## 5. The robot on the cable

The page notices a robot on USB by itself, without a click, once the browser has been allowed
to use it (Web Serial remembers granted devices: `getPorts()`, and the `connect` event). It
asks the robot only `hello` (read-only) and shows a strip on the matching card:

```
│ ⏚ On this computer's USB:  [ Update firmware ]  [ Write apps ]  [ Permissions ] │
```

- An unknown robot: "A robot is plugged in that is not yours yet: **Set it up**" (§3).
- Another account's robot: "This robot belongs to another account" (nothing else offered).
- Nothing changes the robot without a click; a new start app still asks on its screen.
- Not Chrome or Edge, or a phone: no strip; a line says USB needs Chrome or Edge on a computer.

## 6. What the manager remembers

Per robot, so the page can be smart about it:

| What | From |
|---|---|
| firmware version, original firmware recorded or not | the setup page's `hello` (USB) |
| apps, start app, their version; which version the robot has | the manager; the robot reports its version when it connects (the app passes it on) |
| remote changes allowed, asks before the start app changes | the USB setup (the robot holds the real setting) |
| online, last seen, on which app | the apps' robot-auth calls |
| history | every change here and every setup over USB |

## 7. Changes to the plumbing (small)

- Robots report the version of their app list in their Register labels (`apps_ver`); the app
  passes it on in robot-auth, so the manager knows "on the robot ✓".
- robot-auth also tells the manager the app and the robot's firmware (for status and "update
  available").
- The setup page's work (Web Serial, flashing, provisioning) becomes a module the one page uses.

## 8. Decided

1. **A name for the robot** ("Ema's robot"), editable on the card; the id stays in the details.
   Shown on the robot's QR screen later.
2. **Family** (others who may open the robot's apps): later; pairing by the QR code works today.
3. **Firmware updates without USB** (OTA): later; the card says "update over USB".
