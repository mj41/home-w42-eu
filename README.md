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
> **Want more?** Ask in the [issues](https://github.com/mj41/home-w42-eu/issues), and ideally [sponsor mj41](https://github.com/sponsors/mj41) on GitHub:
> mj41 codes for attention food.

## Read in this order

1. [Vision](docs/vision.md): the problem, what we build, what it should feel like.
2. [Use cases](docs/use-cases.md): **the main driver**: who needs what, and when it works.
3. [Principles](docs/principles.md): the binding rules.
4. [Architecture](docs/architecture.md): home node (web/API server, controller
   server, event hub), light clients, adapters, apps, loops, AI agents, trust and
   access control, relay.
5. [Device wire protocol](docs/wire-protocol.md): the reference for how devices,
   adapters and agents connect.
6. [Stackchan](docs/implementations/stackchan.md): the first device family, with
   its apps and the TPBot car.
7. [Fit and roadmap](docs/fit-and-roadmap.md): how the existing repos fit, the gaps,
   and the next stages.

Ideas for devices, adapters and loops (old phones, a Roomba, lawn mowers, Home
Assistant, a private GPS app, cameras and plate OCR, Wi-Fi presence, calendars,
NFC and QR) are in [home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas).

## The repos today

Each repo is independent. A home is made of the repos its user includes.

| Repo | Role |
|---|---|
| [StackChan fork, branch `embody-mj41`](https://github.com/mj41/StackChan/tree/embody-mj41) | Stackchan firmware with Embody Mode, a light client; how to set up a robot: [SETUP.md](https://github.com/mj41/StackChan/blob/embody-mj41/firmware/main/apps/app_embody_mode/SETUP.md) |
| [stackchan-server](https://github.com/mj41/stackchan-server) | Go implementation of the wire protocol (`wire` package), the full Stackchan dashboard, the relay at `chan.w42.eu` |
| [stackchan-pet](https://github.com/mj41/stackchan-pet) | the pet app (a Tamagotchi for kids) |
| [sbot](https://github.com/mj41/sbot) | grows into the home node: web/API server, event hub, controller server; the cockpit app |
| [tpbot-ble](https://github.com/mj41/tpbot-ble) | micro:bit firmware for the TPBot car, laptop tool and bridge |
| [stackchan-mj](https://github.com/mj41/stackchan-mj) | Stackchan working notes, hardware coverage, trust design, build and run scripts |
| [home-w42-eu-ideas](https://github.com/mj41/home-w42-eu-ideas) | ideas for devices, adapters, apps and loops |

How they fit together and what comes next: [Fit and roadmap](docs/fit-and-roadmap.md).

## Status

Design draft, 2026-10-02, after the first working proofs of concept (Stackchan with
three apps, the TPBot car over BLE, the sbot cockpit with joystick and safety stop).

## License

Apache License 2.0, see [LICENSE](LICENSE).
