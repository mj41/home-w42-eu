# Accounts on w42.eu: sign-in providers, linked sign-ins, tiers

**Status:** 2026-10-08. Dex at auth.w42.eu with GitHub and Google (Microsoft prepared, not
enabled); a person (user) with one or more sign-ins in the manager (sm.w42.eu); tiers in
s-w42-eu-raw.

## 1. Providers

Dex (auth.w42.eu) federates the providers; the sites (sm.w42.eu, raw.sa.w42.eu and later ones) talk only to
Dex. Order:

1. GitHub, Google: done.
2. **Microsoft, prepared, not enabled:** Dex's `microsoft` connector and an app registered in Microsoft Entra
   with the redirect `https://auth.w42.eu/callback`. Tenant `consumers` (personal accounts) to
   start; `common` adds work and school accounts. In a multi-tenant sign-in, any tenant's admin
   can set a user's e-mail ("nOAuth"), so **an e-mail from Microsoft never grants anything**:
   admin and tiers by e-mail count only e-mails verified by GitHub or Google.
3. Others (e.g. X) only when someone needs them; X's OAuth gives no e-mail, so such an
   account could not be matched by e-mail in tiers.

Each provider's verified e-mail and its own user id come through Dex (scope `federated:id`:
`federated_claims {connector_id, user_id}`).

## 2. One person, several sign-ins

Dex's issuer and subject differ per provider, so each sign-in is its own identity. The manager
joins them:

- **A user** has an id of our own (`u-` and 16 hex digits) and a list of **sign-ins** (Dex's
  issuer and subject, provider, its user id, login, verified e-mail). The user id is the account
  key everywhere: robots' and linked homes' owner in the manager, and the account every app gets
  through the one sign-in (s-w42-eu-raw's `sso`). A sign-in never owns anything.
- **Adding a sign-in:** signed in, "Your sign-ins" on the manager's page runs the login again for
  the chosen provider; the new sign-in joins the user. A sign-in that already belongs to another
  user brings that whole user along (its robots and homes): signing in with both in one browser
  proves both are this person's. This matters most at the switch to users: every sign-in from
  before became a user of its own.
- **Same e-mail is not enough to join** by itself: the person must sign in with both (an e-mail
  address at a provider can be someone else's later).
- The apps follow at their regular checks (a minute at most): a session's account is replaced
  with the manager's current one, and a robot's owner comes with the robot's next report.
- Later: removing a sign-in (at least one stays).

## 3. Tiers

Tiers set limits, not access. Who is in which tier and each tier's limits: the
[tiers table](https://github.com/mj41/s-w42-eu-raw#robots-set-up-by-a-manager-and-sign-in)
in the s-w42-eu-raw readme.

Rate limiting is the main use: commands per second, open media streams, 640×480 video, robots
an account may add. Hitting a limit says how to get more: tier 5 is asked to sign in, tier 4 to
become a sponsor. The list of tiers 1–3 is a plain file in a private config repository, by
e-mail (`email:`), GitHub login (`github:`) or GitHub user id (`github-id:`). A database comes when the file is not
enough (around a hundred people at once is fine with the file and memory).
