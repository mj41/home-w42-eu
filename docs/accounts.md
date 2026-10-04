# Accounts on w42.eu: sign-in providers, linked sign-ins, tiers

**Status:** 2026-10-04, design. Today: Dex at auth.w42.eu with GitHub and Google (Microsoft
prepared, not enabled); one account per sign-in; tiers in stackchan-server.

## 1. Providers

Dex (auth.w42.eu) federates the providers; the apps (chan.w42.eu and later ones) talk only to
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

## 2. One person, several sign-ins (linking, later)

Today an account is one sign-in: Dex's issuer and subject, which differ per provider, so
signing in with Google gives another account than with GitHub, with its own robots.

Ready for linking:

- **A person** has an id of our own and a list of **sign-ins** (provider + user id). Robots,
  tiers and permissions belong to the person, never to a sign-in.
- **Linking:** signed in, "Add another sign-in" runs the login again; the new sign-in joins the
  person unless it already belongs to someone else (then it is refused; merging two people is
  done by an admin). Unlinking keeps at least one sign-in.
- **Same e-mail is not enough to link** by itself: the person must sign in with both
  (an e-mail address at a provider can be someone else's later).
- In stackchan-server this replaces `Account.Key` as the owner of added robots by the person
  id (principle 27: the stored accounts start over).

## 3. Tiers

Tiers set limits, not access. Who is in which tier and each tier's limits: the
[tiers table](https://github.com/mj41/stackchan-server#other-peoples-robots-sign-in-and-add-your-own)
in the stackchan-server readme.

Rate limiting is the main use: commands per second, open media streams, 640×480 video, robots
an account may add. Hitting a limit says how to get more: tier 5 is asked to sign in, tier 4 to
become a sponsor. The list of tiers 1–3 is a plain file in a private config repository, by
e-mail (`email:`), GitHub login (`github:`) or GitHub user id (`github-id:`). A database comes when the file is not
enough (around a hundred people at once is fine with the file and memory).
