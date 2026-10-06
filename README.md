# home-w42-eu

A **local first, security and privacy first platform for a home**: Go servers on a
small machine at home around one event hub, every device (new or old) a light client
of them, and loops and controllers that AI helps you write and you approve.

> Nothing leaves the home unless its owner signs that it may.

> **Design drafts and proofs of concept, vibe coded.** Written with AI agents. The code in the
> repos below is tested on real hardware at home, but neither the code nor its security has
> been reviewed by humans. Use it on your own network, and don't trust it with anything
> private yet.
>
> **Early stage: no backward compatibility.** Protocols, APIs, file formats and stored settings
> change when something better comes along, without migrations: update the robot's firmware
> and the servers together.
>
> **Want more?** Ask in the [issues](https://github.com/mj41/home-w42-eu/issues), and ideally [sponsor mj41](https://github.com/sponsors/mj41) on GitHub:
> mj41 codes for attention food.

## Read in this order

1. [Vision](docs/vision.md): the problem, what we build, what it should feel like.
2. [Use cases](docs/use-cases.md): **the main driver**: who needs what, and when it works.
   - [Personas and agents](docs/personas.md): the people and programs the use cases are for
     (the owner, the kids, family far away, guests, AI agents, loops, the relay…), what
     matters first to each and what each may do.
3. [Principles](docs/principles.md): the binding rules.
4. [Architecture](docs/architecture.md): home node (web/API server, controller
   server, event hub), light clients, adapters, apps, loops, AI agents, trust and
   access control, relay.
5. [Device wire protocol](docs/wire-protocol.md): the reference for how devices,
   adapters and agents connect.
   - [End-to-end encryption](docs/e2ee.md): a device and its enrolled browsers, through a
     relay that carries only ciphertext.
   - [Device API](docs/device-api.md): sensors in real units, actuators as named parts
     (`left1`, `yaw`), after Linux's sysfs and IIO.
   - [Device setup](docs/device-setup.md): firmware, connecting over USB, one page with
     the person's consent, and what the robot confirms itself.
   - [App catalog](docs/app-catalog.md): the apps a home's devices can use, one directory
     per app, shared by the home's servers.
   - [Device storage](docs/device-storage.md): per-app folders under one root
     (`/user/embody`, and `/sdcard/embody` with a microSD card), uninstall and remove-all.
6. **w42.eu services and releases:**
   - [Stackchan sites](docs/stackchan-sites.md): sm.w42.eu (the manager: robots, the apps
     you approve, setup) and one host per app (`raw.sa.w42.eu`, `pet.sa.w42.eu`, …).
     Deployed 2026-10-04.
   - [Accounts](docs/accounts.md): sign-in providers (GitHub and Google; Microsoft
     prepared), linked sign-ins, tiers and rate limits.
   - [Analytics](docs/analytics.md): counting visits to the public sites with GoatCounter, no
     cookies, nothing from homes.
   - [Independent rebuild](docs/independent-rebuild.md): a third verification of releases on
     a short-lived cloud VM (Linode): Espressif's toolchain built from source with Guix host
     tools, a certified key of its own, a signed audit log.
7. [Stackchan](docs/implementations/stackchan.md): the first device family, with
   its apps and the TPBot car.
8. [Fit and roadmap](docs/fit-and-roadmap.md): how the existing repos fit, the gaps,
   and the next stages.

Ideas for devices, adapters and loops (old phones, a Roomba, lawn mowers, Home
Assistant, a private GPS app, cameras and plate OCR, Wi-Fi presence, calendars,
NFC and QR) are in [home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas).

## The repos today

Each repo is independent. A home is made of the repos its user includes.

| Repo | Role | License |
|---|---|---|
| [StackChan fork, branch `embody-mj41`](https://github.com/mj41/StackChan/tree/embody-mj41) | Stackchan firmware with Embody Mode, a light client; how to set up a robot: [SETUP.md](https://github.com/mj41/StackChan/blob/embody-mj41/firmware/main/apps/app_embody_mode/SETUP.md) | MIT (the firmware, as upstream) |
| [s-w42-eu-manager](https://github.com/mj41/s-w42-eu-manager) | the Stackchan manager at `sm.w42.eu`: sign-in, your robots, the apps you approve, one-click setup over USB, a token per robot per app | MIT |
| [s-w42-eu-raw](https://github.com/mj41/s-w42-eu-raw) | Go implementation of the wire protocol (`wire` package), the full Stackchan dashboard and relay at `raw.sa.w42.eu`, the `robotauth` package the apps share | MIT |
| [s-w42-eu-pet](https://github.com/mj41/s-w42-eu-pet) | the pet app (a Tamagotchi for kids), at `pet.sa.w42.eu` | MIT |
| [s-w42-eu-sbot](https://github.com/mj41/s-w42-eu-sbot) | grows into the home node: web/API server, event hub, controller server; the cockpit app | Apache-2.0 |
| [tpbot-ble](https://github.com/mj41/tpbot-ble) | micro:bit firmware for the TPBot car, laptop tool and bridge | Apache-2.0 |
| [home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas) | ideas for devices, adapters, apps and loops | Apache-2.0 |
| [mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved) | the signed hashes of each release, per builder, and the tool that checks them ([device setup](docs/device-setup.md) §6.2) | Apache-2.0 |
| [mj41cz-rebuilds](https://gitlab.com/mj41cz/mj41cz-rebuilds) | independent rebuilds of releases on short-lived cloud machines, with a signed audit log ([independent rebuild](docs/independent-rebuild.md)) | Apache-2.0 |

How they fit together and what comes next: [Fit and roadmap](docs/fit-and-roadmap.md).

## Status

Design draft with working proofs of concept, 2026-10-06:

- **Stackchan** with three apps (the dashboard, the pet, the sbot cockpit) and the TPBot car
  over BLE.
- **sbot** with the event hub (JetStream) and the safety stop; its controller server runs
  the `frown` loop live.
- **The Stackchan manager**, at [sm.w42.eu](https://sm.w42.eu) and at home on your own
  computer: one page for your robots, one-click setup over USB, the apps you approve for each
  robot with a token per robot per app. Each robot keeps a live connection to its one manager
  ([the robot's manager](docs/manager-channel.md)): switch apps, restart, change apps from the
  page, see a question on its screen. A home manager links up to sm.w42.eu (read-only or full
  control); the manager is optional, and switching is always possible on the robot. The apps:
  the raw dashboard at `raw.sa.w42.eu` (robots private by default, end-to-end encryption) and
  the pet at `pet.sa.w42.eu`. `s.w42.eu` is the index.
- **Firmware `embody-v0.5.1`** released, rebuilt to the same bytes by GitHub Actions, the
  owner's laptop and a cloud rebuild, signed by the owner in
  [mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved).

What comes next: [Fit and roadmap](docs/fit-and-roadmap.md).

Stack-chan (スタックチャン) is a registered trademark of Shinya Ishikawa; this project is independent and only made to work with [Stack-chan](https://github.com/stack-chan/stack-chan) robots.

## License

Apache License 2.0, see [LICENSE](LICENSE).
