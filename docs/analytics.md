# Analytics for w42.eu

**Status:** 2026-10-04, design; nothing deployed. Asked for: friendly, local, open source, no
Java, ideally Go or Rust.

## 1. What we want to know

- **Pages:** which pages of the public sites (chan.w42.eu with its setup page, the docs; the
  flasher once it is its own page, planned) are read, from where people come (referrer),
  screen sizes, countries. Counts, not people.
- **The setup path:** how many start "Set up my robot", how many finish, where they stop (no
  USB permission, flashing failed, not confirmed on the robot). These are events the pages send.
- **The servers:** robots online, sign-ins per [tier](accounts.md#3-tiers), how often limits
  are hit: from the servers' own counters, not from browsers.

Never: what a robot sees, hears or does; anything from a home server; anything that identifies
a person (no cookies, no stored IP addresses, no fingerprinting). Principle 1: data stays at
home; the public sites are the only thing counted.

## 2. The tool: GoatCounter

[GoatCounter](https://github.com/arp242/goatcounter): Go, open source (EUPL-1.2), one binary with
SQLite or PostgreSQL, no cookies and no personal data by design (no consent banner needed), a
small script or a pixel, events, an API, a clear dashboard. Runs on our own cluster, so no
third party sees the visits.

Not chosen: Plausible (Elixir and ClickHouse: heavy for our size), Umami (Node.js), Matomo
(PHP), anything hosted by others. A Rust option (e.g. Liwan) is younger; GoatCounter has years
of use behind it.

## 3. How

1. **stats.w42.eu:** GoatCounter in the cluster, its dashboard only for the owner (sign-in
   through auth.w42.eu in front of it); SQLite on a small volume.
2. **Pages:** the GoatCounter script on chan.w42.eu's public pages (today `/setup` flashes
   the firmware; later also the flasher on GitHub Pages), served from stats.w42.eu (no CDN);
   the setup page sends its steps as events (`setup/start`, `setup/flashed`,
   `setup/confirmed`, `setup/failed/<step>`).
3. **Servers:** s-w42-eu-raw counts (robots online, sessions per tier, 429s per tier) and
   sends hourly totals to GoatCounter's API, or shows them on an admin page; no per-session
   data leaves the server.
4. **Home servers are never counted.** The script is only in the pages chan.w42.eu serves with
   `-public-url` set to a w42.eu address.

Deploying it waits until the cluster's configuration may change again.
