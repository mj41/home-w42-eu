# App catalog: the apps a home's devices can use

**Status:** 2026-10-03, design. Today each server lists other apps with `-offer` flags; this
document replaces them with one catalog per home: a directory with one directory per app. Setting devices up: [Device setup](device-setup.md).

## 1. The problem

A Stackchan can switch between apps, each a server: the dashboard (s-w42-eu-raw), the pet
(s-w42-eu-pet), the cockpit (sbot), a public relay (raw.sa.w42.eu). Today:

- every server repeats the others in its own flags (`-offer Pet=ws://…,<token file>`), so
  adding an app means editing several command lines and restarting;
- the setup page can only pin what that one server happens to offer;
- nothing says which apps stay at home and which send data out (raw.sa.w42.eu), or which
  devices may join which app (principle 22: the owner decides).

## 2. One catalog, not one config

Not everything belongs in one file. Four kinds of settings, each with one home (principle 13):

| What | Where | Who changes it |
|---|---|---|
| **App catalog:** which apps the home has, where they are, who may join them | one directory per home, with one directory per app, read by every server of the home; later the node, each app signed by the owner key | the owner |
| **Server settings:** how one server runs (listen address, TLS, sign-in, state file) | that server's flags or its own config | whoever runs it |
| **Secrets:** robot tokens, OIDC secrets | files of their own (an app's token in its directory, mode 0600), never inside a config file | the owner |
| **Device state:** the servers a robot accepted, its pin, autostart, Wi-Fi | the robot (NVS) | the owner, at the robot or through an app |

So: **one catalog** for the apps, shared by all servers of a home, next to each server's own
settings, and **one directory per app** in it, like `/etc/…/conf.d`: an app is added or removed
as a whole, carries its own token and, later, its own signature. On w42.eu the manager
(sm.w42.eu) has its own catalog of the public apps and gives each robot a token of its own per
app ([Stackchan sites](stackchan-sites.md)); it never hands out a home's tokens.

## 3. The directory

`~/.config/w42eu/apps/` (`$XDG_CONFIG_HOME`; `/etc/w42eu/apps/` for a node run as a service; the
path is a flag, `-apps`, and every server of the home points to the same directory and reads
it again when something in it changes):

```
~/.config/w42eu/apps/
├── dashboard/
│   ├── app.yaml        name, addresses, devices, start
│   └── token           the robot token devices use for this app (0600)
├── pet/
│   ├── app.yaml
│   └── token -> ../dashboard/token     apps that share a token link to it
├── cockpit/
│   ├── app.yaml
│   └── token -> ../dashboard/token
└── raw-w42/
    ├── app.yaml
    └── token           this robot's token for raw.sa.w42.eu, from sm.w42.eu
```

`dashboard/app.yaml`:

```yaml
name: Dashboard
url: ws://192.168.1.10:8765           # where devices connect
web: http://192.168.1.10:8765         # where people open it (default: url with http)
devices: [stackchan]                  # device families (or ids) that may join
start: true                           # the default app to start with at setup
```

`raw-w42/app.yaml`:

```yaml
name: raw.sa.w42.eu
url: wss://raw.sa.w42.eu
devices: [stackchan-0a1b2c3d4e50]     # only this robot
leaves_home: true                     # data goes through a public relay: shown as such
```

- The directory name is the app's `id`: stable, while the name can change.
- `enabled: false` keeps an app in the catalog without offering it.
- `leaves_home` apps are marked in the setup page and on the robot's list, and are never the
  default start unless the owner picks them.
- No `token` file: devices join without one (an app that does not check), or the app is listed
  for people only.
- Later, per app: `app.yaml.sig` (the owner key's signature), `icon.png`, a short `README`.
- Order: by directory name, `start: true` first.

## 4. Who uses it

| User | Uses the catalog for |
|---|---|
| Every server of the home | `ServerOffer` to devices that may join (replaces `-offer`) |
| The connect page ([Device setup](device-setup.md) §3.4) | the app list and the default start; the robot gets every allowed app with its token |
| The robot | nothing directly: it keeps its own accepted list, pin and autostart (device state) |
| Apps | links to each other ("open in the pet") |

The robot's list stays the device-side truth: the catalog *offers*, the owner (at the robot,
on the setup page, or in an app) *accepts* and *pins*.

## 5. Managing it

1. **Now:** the directories, made by hand (`mkdir`, an editor, `ln -s` for a shared token);
   they replace the `-offer` flags (no backward compatibility, principle 27).
2. **Then:** an "Apps" page on the home server: add, remove, rename, which devices, the
   default start; it writes the directories. An app can also bring its own directory (its
   installer creates it), listed as *new* until the owner approves it (an `approved` file).
3. **Target (architecture §5):** devices connect only to the node, which routes them to apps;
   the catalog lives in the node's registry and is signed by the owner key, and devices verify
   it (principle 3), so a server cannot offer an app the owner did not list.

4. **Later: a hub of apps from others.** A public registry where other people publish apps
   would need security checks (review, signatures, permissions an app asks for, reports) before
   anything from it is offered to a robot. Not now (decided 2026-10-04): the catalog lists only
   our own apps and the ones the owner adds by hand.

## 6. Decisions

2026-10-03:

- **Tokens:** one per app now, in the app's directory (`token`, mode 0600), shared between
  apps by a symlink, as the home's robot token is today. Target: one per device and app,
  issued by the node's registry, then device certificates (principle 3).
- **Format:** YAML for `app.yaml`, written by people; the signature (later) covers the file's
  bytes, so the format does not matter for signing.
- **Approving an app that brought its own directory:** an `approved` file in its directory,
  written by the owner or the Apps page; until it is there the app is listed as *new* and
  offered to no device. Once signing exists, `app.yaml.sig` replaces it.
- **Where:** `~/.config/w42eu/apps/`, or `/etc/w42eu/apps/` for a node run as a service.
