# Storage on a device: per-app folders under one root

**Status:** 2026-10-04, design (proposal, two questions open in §5; microSD and "no quotas"
decided). For Stackchan's Embody Mode
first; other devices with a file store follow the same layout.

## 1. Today

A Stackchan keeps files that servers upload (pictures, sounds) in its `userdata` partition (FAT,
about 1.9 MB), mounted at `/user`, under names the server chooses (`pet/snd/eat.wav`). Settings
are in NVS (namespaces `embody`, `embody_e2e`). Problems:

- every server sees and may delete every file; the pet keeps to `pet/` by convention only;
- nothing removes an app's files when the app is gone;
- nothing removes all of Embody Mode's data at once.

## 2. One root, laid out like Linux

Everything Embody Mode stores goes under one folder, **`/user/embody`**, after the Linux filesystem
hierarchy:

```
/user/embody/
├── var/lib/<app>/     an app's data: what its server uploads (pictures, sounds)
├── var/cache/<app>/   an app's cache: may be deleted any time, e.g. when space runs out
├── etc/               Embody Mode's own configuration files (later; settings stay in NVS for now)
└── tmp/               uploads in progress (today /user/.upload.tmp)
```

- **Remove all:** delete `/user/embody` (and `/sdcard/embody`, §2.1) and erase the NVS namespaces
  `embody` and `embody_e2e`. Everything else on the robot (other apps of the launcher, Wi-Fi)
  stays.
- **Uninstall an app:** delete `var/lib/<app>` and `var/cache/<app>` (on both).

### 2.1 A microSD card, when there is one

The CoreS3 has a microSD slot. With a card in it, Embody Mode keeps the same tree on the card,
**`/sdcard/embody`** (`var/lib/<app>`, `var/cache/<app>`, `tmp`), for more data and caches (photos,
recordings, sounds, a cache of pictures). An app sees both its folders: small things it always
needs (its sounds) in the internal one, which is there with or without a card; large or
optional things on the card. Without a card, only the internal 1.9 MB.

- The file commands take a place: `"store": "internal"` (default) or `"sd"`; `assets` lists
  both, with free and total space of each. A command for `sd` without a card fails with
  "no microSD card".
- The card is FAT (as cards come); the robot does not format it unless asked (a "Format the
  card" in its settings, confirmed on the screen). Files outside `/sdcard/embody` stay untouched.
- Taking the card out while running: the app's `sd` commands fail until it is back; nothing is
  lost on the internal store.

### 2.2 Space: no quotas, first come, first served

No per-app quota (decided 2026-10-04): an app may use whatever space is free, internal and on
the card; when it is full, an upload fails with "no space" and the app decides what to delete
(or its cache is deleted first, `var/cache/`). `assets` shows each app what it uses and what is
free. A quota can come later if apps start crowding each other out.

## 3. An app sees only its own folder

`<app>` is the app connected right now (the server the robot talks to). Its file commands work
inside its folders only:

| Command | Inside |
|---|---|
| `assets` (list), upload (binary 0x11), `asset_delete` | `var/lib/<app>/` |
| `sprite {asset}`, `play {asset}` | `var/lib/<app>/` |
| `app_data_remove` (new): "uninstall me" | removes `var/lib/<app>` and `var/cache/<app>` |

Names stay relative, as today (`snd/eat.wav`); `..` and absolute paths are refused as today.
All apps share the same space, first come, first served (§2.2).

## 4. Uninstall and remove-all, where

- **On the robot:** removing a server from the robot's list (QR screen, or `server_remove` from a
  browser) offers to remove its app's data too; a "Remove all Embody data" in the robot's settings,
  with a confirmation on the screen.
- **From the server:** `app_data_remove` (its own data only).
- **Over USB:** `{"op":"reset"}` removes all (the setup page's "Start over"), confirmed on the
  screen like a new default server.

## 5. Questions

1. **The app's name (`<app>`).** Proposed: the server sends it when it accepts the robot (an
   optional `app` field in `Accepted`, e.g. `pet`, `sbot`, `dashboard`; letters, digits, `-`, up to
   32), a v1 addition ([wire protocol](wire-protocol.md) §10); a server that does not send it gets
   its host and port (`192.168.1.10-8770`). Two servers sending the same name share a folder: fine
   for the same app on two hosts, wrong for a malicious server; with signed app catalogs
   ([app catalog](app-catalog.md)) the name comes from the catalog instead. OK?
2. **Your dashboard and other apps' files.** Proposed: no exception; every app sees its own folder,
   the dashboard too. The robot's own "Remove all" and per-app uninstall cover cleaning up. Or
   should the owner's dashboard see (and delete) every app's files?
3. ~~Today's files.~~ Decided by principle 27 (no backward compatibility): the new firmware
   starts with an empty `/user/embody`, and files outside it are deleted on its first start; the
   pet uploads its files again by itself, anything else is uploaded again by hand.
