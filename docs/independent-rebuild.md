# Independent rebuild: a third verification, in the cloud, with its own key

**Status:** 2026-10-04, design. Part of the release hardening in
[Device setup](device-setup.md) §6. Two verifications exist today: GitHub Actions builds a release,
and the owner's laptop rebuilds it (with Espressif's compiler, with our own compiler, and with
Guix's host tools: the same bytes). This adds a third: a short-lived cloud machine that bootstraps
everything from Guix's seed, rebuilds the release, signs what it saw with a key that never leaves
it, and is deleted afterwards. Its volume, with the Guix store, is kept for the next runs.

## 1. Why

- **Another party's computer:** neither GitHub (where the release is built) nor the owner's
  laptop (where it is approved) nor Akamai (where chan.w42.eu runs).
- **From the seed:** the VM builds its whole host toolchain from Guix's bootstrap seed
  (`--no-substitutes`), so it trusts no prebuilt compiler, then Espressif's toolchain from source
  (`firmware/toolchain/`), then the firmware.
- **Reproducible by anyone:** the same scripts, the same pinned Guix channel, any cloud account.
- **A record, not a claim:** every step, version and hash goes into an audit log, committed and
  signed on the VM, published in a public repository.

## 2. Keys

| Key | Where | Lifetime | Signs |
|---|---|---|---|
| **Owner key** | the owner's laptop (later a hardware key) | long | the VM key's certificate; the release approval |
| **VM key** (SSH, Ed25519) | created on the VM, in memory (tmpfs), never copied out | the VM's life | every commit of the audit log, the VM's `SHA256SUMS` |
| **VM certificate** | the owner key signs the VM key's public half (`ssh-keygen -s owner_key -I rebuild-<release>-<date> -n git -V +12h vm_key.pub`) | 12 hours | — |

Git checks SSH signatures against an `allowed_signers` file that can name a **certificate
authority**: one line with the owner's public key and `cert-authority,namespaces="git"`. Every
VM commit then verifies against the owner key, inside the certificate's 12 hours, while the VM's
private key is gone once the VM is deleted.

## 3. The run

From the owner's laptop, one command (`rebuild-cloud <release>`):

1. **Create:** with its own cloud API token (a file of its own, not the GitOps repo's, limited to
   this project's machines and volumes): a VM (e.g. 32 vCPU) with the project's volume attached
   (created on the first run) for the Guix store; cloud-init installs Guix from its signed binary
   release and nothing else.
2. **Certify:** fetch the VM's new public key over SSH (host key pinned from the provider's
   console output), sign its certificate with the owner key, put the certificate on the VM.
3. **Publish access:** add the VM's key as a write deploy key of the audit repository (GitLab API,
   the owner's token from a local file), only for this run.
4. **Build, on the VM:**
   - Guix pinned by `channels.scm` (`guix time-machine`), `--no-substitutes`: the host tools from
     the seed (hours; the volume holds the store);
   - Espressif's toolchain from source at `/builds/idf/crosstool-NG`;
   - the release firmware at the release's commit;
   - its `SHA256SUMS`, compared with GitHub's release and the owner's.
5. **Record, on the VM:** an audit log (each step with its start, end, command, versions, the
   store paths and hashes of the host tools, the toolchain's and the firmware's hashes), committed
   and pushed step by step, each commit signed by the VM key. The final commit adds the VM's signed
   `SHA256SUMS`; a merge request adds them to `mj41cz-approved` as one more builder.
6. **Delete the VM:** remove the deploy key, detach the volume and delete the VM; the run's last
   log line (from the laptop) records it, with the provider's API answers. The **volume is kept**:
   its Guix store serves the next runs. It holds no key (the VM key lives in memory only) and no
   token.

## 4. The audit repository

A new public repository on GitLab, `mj41cz-rebuilds` (not GitHub, not Akamai), separate from the
GitOps repository and its tokens:

```
mj41cz-rebuilds/
├── allowed_signers                  the owner key as cert-authority
├── tool/                            the verification tool (§5), its channels.scm and manifest.scm
└── stackchan-embody/embody-v0.2.0/2026-10-05/
    ├── run.json                     provider, region, machine type, image, channel commit, times
    ├── log/01-guix-bootstrap.txt …  every step's output (trimmed), each in its own signed commit
    ├── host-tools.txt               store paths and hashes of the host tools built from the seed
    ├── toolchain-SHA256SUMS         Espressif's toolchain as built on the VM
    ├── SHA256SUMS(.sig)             the firmware, signed by the VM key
    └── post.md                      the blog post, written afterwards by the owner
```

Commits on the run's branch are signed by the VM key (certified by the owner); the owner's later
commits (the post) by the owner key.

## 5. The verification tool

For anyone, Guix-based so its own tools are pinned and reproducible:

```bash
guix time-machine -C tool/channels.scm -- shell -m tool/manifest.scm -- \
    ./tool/verify stackchan-embody/embody-v0.2.0
```

It clones `mj41cz-rebuilds` and `mj41cz-approved`, downloads the release's files from GitHub, and
checks:

- every audit commit's signature (the VM certificate against the owner key, within its validity);
- the firmware files' SHA-256 against every builder's `SHA256SUMS` (CI, the owner, the cloud VM)
  and their signatures;
- the CI attestation (Sigstore, `gh attestation verify`) and the owner's timestamp;
- that the audit log is complete (every step present, in order) and consistent (the hashes it
  recorded are those in `SHA256SUMS`).

The tool is a small Go program; the Guix manifest pins Go, git, OpenSSH, OpenSSL and the GitHub
CLI.

## 6. Upgrades: one change at a time

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

## 7. Decisions and open points

- **Provider:** any with an API and cloud-init; the scripts keep the provider in one small part.
  For independence from Akamai (chan.w42.eu) and GitHub (CI), AWS or Google Cloud rather than
  Linode; Linode works too, with a token of its own.
- **Cost:** hours of a large VM per run (the seed bootstrap dominates), a few euros.
- **Run per release, or per toolchain change?** The seed bootstrap and the toolchain change rarely;
  a release only needs the firmware step. Proposed: the full run for every new toolchain or Guix
  channel, a short run per release, reusing the volume's Guix store and toolchain. A short run first
  checks the volume against the last full run's audited hashes (`guix gc --verify=contents`, the
  toolchain's `SHA256SUMS`), so a changed volume is caught.
- **A fresh volume** for a full run from the seed, so nothing is inherited; the old volume is
  deleted only after the new one is verified.
