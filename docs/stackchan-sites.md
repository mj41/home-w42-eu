# Stackchan on w42.eu: a manager and separate apps

**Status:** 2026-10-04, design for review. Replaces chan.w42.eu, which is removed (no backward
compatibility, [principles](principles.md) 27).

## 1. Hostnames

`s.w42.eu` and `s*.w42.eu` are Stackchan's; home.w42.eu is a different project.

| Host | What |
|---|---|
| `s.w42.eu` | index and project page (static) |
| `sm.w42.eu` | **the Stackchan manager:** sign-in, your robots, the apps you approve for them, one-click setup over USB, their tokens, the official firmware |
| `raw.sa.w42.eu` | the raw dashboard: every sensor and command (today's chan.w42.eu dashboard), an app like any other |
| `pet.sa.w42.eu` | the pet ([stackchan-pet](https://github.com/mj41/stackchan-pet)) |
| `dev.sa.w42.eu` | the higher-level API for developers and AI agents ([device API](device-api.md)), planned |
| `*.sa.w42.eu` | more apps, one host each |

**Every app has its own host,** because the browser isolates by origin (host), not by path:
pages of one app cannot read another's storage (e.g. the end-to-end encryption keys), cookies
or sign-in, and cannot call its API as the user. One wildcard certificate covers `*.sa.w42.eu`.

## 2. The manager sets up the apps you approve

1. You sign in on sm.w42.eu (auth.w42.eu: GitHub, Google).
2. You pick the apps for your robot from the manager's catalog (raw, pet, …) and the one it
   starts with.
3. One click with the robot on USB: the manager installs the official firmware (if needed) and
   writes every approved app into the robot, each with **a token of its own** (`provision`
   with `servers` and `pin`); the robot asks on its screen before its start app is set.
4. The robot connects to its start app; the QR screen switches between the approved apps.

Removing an app on sm.w42.eu revokes its token at once.

## 3. Robot tokens: issued by the manager, checked by each app

- One token **per robot per app**; the manager keeps only the hash.
- An app checks a robot's token when the robot connects: it asks the manager
  (`POST sm.w42.eu/api/robot-auth {app, robot, token}`, authenticated as that app) and gets the
  robot's owner and whether it is private or public. The answer is cached for a few minutes.
- Apps hold no robot tokens and no account database. Later the manager signs the tokens and apps
  verify them offline ([trust design](https://github.com/mj41/stackchan-mj/blob/main/docs/design.md)).

## 4. People, pairing, privacy

- One sign-in for all sites: each site is its own client of auth.w42.eu, and an account is the
  same everywhere (issuer and subject).
- A robot is private to its owner by default in every app: the app learns the owner from the
  manager and pairs only them (and whom they allow). End-to-end encryption stays per app.
- Tiers and limits ([accounts](accounts.md)) apply per app, decided by the manager.

## 5. Steps

1. **DNS and certificates:** `s.w42.eu`, `sm.w42.eu`, wildcard `*.sa.w42.eu` (GitOps).
2. **Split stackchan-server into two programs** in its repo: `stackchan-manager` (sign-in,
   robots, the app catalog, setup, firmware, `robot-auth`) and the raw app (robot connections,
   pairing, the dashboard, the end-to-end relay), which checks tokens with the manager. Deploy
   them as sm.w42.eu and raw.sa.w42.eu.
3. **pet.sa.w42.eu:** the pet checks tokens with the manager too (a small client both apps share).
4. **s.w42.eu:** the index page.
5. **Remove chan.w42.eu:** deployment, DNS, docs. Robots are set up again on sm.w42.eu.
6. **dev.sa.w42.eu** once the device API exists.

## 6. Open questions

1. **Apps added later:** how a robot learns a newly approved app without USB. Proposed: the app
   it is on asks the manager and sends the robot a `ServerOffer` with the new app and its token.
2. **Tiers across apps:** the manager answers per account, or the apps read the same tiers file.
3. **Home servers:** a home keeps its own servers and [app catalog](app-catalog.md); the public
   manager is for w42.eu. Should a home server be able to use sm.w42.eu for setup?
4. **The pet in public:** its leaderboard photos would be stored on a public server, and its
   parent page has a PIN instead of sign-in. Decide before pet.sa.w42.eu goes live.
