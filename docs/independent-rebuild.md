# Independent rebuild: a third verification, in the cloud, with its own key

**Status:** 2026-10-04, implemented in [mj41cz-rebuilds](https://gitlab.com/mj41cz/mj41cz-rebuilds);
first release run 2026-10-04 (`embody-v0.1.0`, identical bytes). Part of the release hardening
in [Device setup](device-setup.md) §6. GitHub Actions builds a release, and the owner's laptop
rebuilds it (with Espressif's compiler, with our own compiler, and with Guix's host tools: the
same bytes). This adds a third verification: a short-lived cloud machine that builds
Espressif's toolchain from source with Guix host tools, rebuilds the release, signs what it saw
with a key that never leaves it, and is deleted afterwards. Its volume, with the Guix store, is
kept for the next runs.

## 1. Why

- **Another computer:** neither GitHub (where the release is built) nor the owner's laptop
  (where it is approved), but a fresh machine for each run, deleted afterwards.
- **Its own toolchain:** the machine builds Espressif's toolchain from source
  (`firmware/toolchain/`) with Guix's host tools, then the firmware, and compares the bytes with
  the release's.
- **Reproducible by anyone:** the same tool, the same pinned Guix channel, any cloud account.
- **A record, not a claim:** every step, version and hash goes into an audit log, committed and
  signed on the machine, published in a public repository.

## 2. Keys

| Key | Where | Lifetime | Signs |
|---|---|---|---|
| **Owner key** | the owner's laptop, in ssh-agent (later a hardware key) | long | the machine key's certificate; the release approval |
| **Machine key** (SSH, Ed25519) | created on the machine in tmpfs, then held only by an ssh-agent there (the file is shredded); never copied out | the machine's life | every commit of the audit log |
| **Machine certificate** | the owner key signs the machine key's public half for the principal `rebuild-<run>@w42.eu` (`ssh-keygen -U -s owner_key -I rebuild-<run> -n rebuild-<run>@w42.eu -V -5m:+12h`) | 12 hours | — |

Git checks SSH signatures against an `allowed_signers` file that can name a **certificate
authority**: one line `rebuild-*@w42.eu cert-authority,namespaces="git" <owner key>`. Every
commit of a run then verifies against the owner key, inside the certificate's 12 hours, while
the machine's private key is gone once the machine is deleted.

The machine never pushes: it holds no token and no deploy key. At the end of a run the laptop
fetches the audit log from it (a git bundle) into the branch `runs/<run>` of `mj41cz-rebuilds`,
which is then pushed to GitLab; the machine's `SHA256SUMS` go to `mj41cz-approved` as the
builder `cloud`.

## 3. How a run works

The commands, the plans (`smoke`, `full`), the audit log's layout and how to verify a run:
the [mj41cz-rebuilds README](https://gitlab.com/mj41cz/mj41cz-rebuilds).

## 4. Upgrades: one change at a time

A new compiler, Guix channel, ESP-IDF or library version is tried on the VMs **before** any new
firmware is built with it:

1. **Rebuild the last release with the new tools,** from its tag, unchanged. Its published bytes
   are known (CI, the owner, the earlier VM runs).
2. **The same bytes:** the new tools do not change our firmware; recorded in the audit log, and
   the tools become current.
3. **Different bytes:** the difference is the new tools' alone, since the source did not change.
   It is explained (as the build path and the host compiler were in
   [Device setup](device-setup.md) §6.4) and recorded before anything else moves; an unexplained
   difference stops the upgrade.
4. **Only then a new firmware,** built with the accepted tools.

A regression in a new release then has one of two causes, never both at once: the tools (step 3
shows it on the old source) or our code (the tools were already proven on the old source).

## 5. Decisions and open points

- **Provider:** Linode today, with an API token of its own, limited to machines and volumes, not
  the GitOps repository's; the tool keeps the provider in one small part (`internal/cloud`). Linode
  is Akamai, which also runs chan.w42.eu; another provider (e.g. AWS or Google Cloud) would add
  independence.
- **A full run from Guix's seed** (`-seed`): the host tools built on the machine from Guix's
  bootstrap seed (`--no-substitutes`, except Rust) instead of substitutes from the build farms
  checked by `guix challenge`. Implemented, not run yet; hours per run.
- **Cost:** about an hour of a 16-CPU machine per full run; hours with `-seed`.
- **Run per release, or per toolchain change?** The host tools and the toolchain change rarely;
  a release only needs the firmware step. Proposed: the full run for every new toolchain or Guix
  channel, a short run per release, reusing the volume's Guix store and toolchain. A short run first
  checks the volume against the last full run's audited hashes (`guix gc --verify=contents`, the
  toolchain's `SHA256SUMS`), so a changed volume is caught.
- **A fresh volume** for a full run from the seed, so nothing is inherited; the old volume is
  deleted only after the new one is verified.
