# REVIEW_REQUEST — code review of the `fleet` branch (commit 4e91f9d) in cluster-utilities

**Task: adversarial CODE REVIEW.** Find real bugs, unsafe behaviour, and design flaws in the new fleet
tooling. Do not propose new features; judge whether what's here is correct and safe to use.

## Context
cluster-utilities holds the owner's notes and scripts for reaching compute (Alliance HPC clusters, see
AGENT.md) and now their own small fleet of machines over Tailscale: a laptop (`zephyrus-m16`) and a lab
workstation (`lkj008-psy`, ssh alias `lab`). Three more lab workstations (CachyOS already installed) are
about to be onboarded. All hosts run CachyOS (Arch-based). The owner's login shell is **fish** on every
host. No YAML parser (yq / PyYAML) is installed on the hosts, so the YAML files are read with awk.
Password ssh login is intentionally LEFT ENABLED on the lab boxes (owner's decision; do not flag it).

Goal of the branch: "one project, one host". Research repos live on lab machines, desktop/personal ones
on the laptop, infra repos on every host. Repos are moving from `~/code` (legacy) to `~/Projects`.

## Files to review (read them in full)
New:
- `bin/fleet-audit`: run from the laptop. Scans each active host's repo roots (locally, or by piping a
  bash scanner over `ssh -o BatchMode=yes <alias> bash -s`) and flags unsaved work (NOGIT, NOREMOTE,
  UNPUSHED, DIRTY, UNTRACKED, STASH, BEHIND) plus placement problems against `fleet/projects.yaml`.
  The owner will use its output to decide which copies are SAFE TO DELETE, so false negatives (work
  reported as safe when it is not) are the most serious class of bug.
- `bin/fleet-onboard`: run as the user on a NEW machine at its keyboard, then self-elevates with sudo.
  Sets the hostname, pacman-installs packages, enables sshd and tailscaled, applies the MagicDNS fix
  (symlinks /etc/resolv.conf to the systemd-resolved stub), runs `tailscale up` (optional TS_AUTHKEY),
  and installs an embedded fish SSH login menu into the invoking user's `~/.config/fish/conf.d/`.
  It must be idempotent and must never lock the owner out of the machine or its network.
- `fleet/hosts.yaml`, `fleet/projects.yaml`: the awk-parsed data files. Check that the awk in
  fleet-audit reads them as documented (comments, commented-out host stanzas, inline `# …` comments,
  `[?]`, multi-host lists).
- `fleet/README.md`: the onboarding procedure. Check it matches what the scripts actually do.

Modified (the diff is inlined below because the snapshot has no .git):
- `AGENT.md` (one paragraph added in the Setup section pointing to fleet/)
- `lab-access.md` (lab auth: was "password only", now "key + password")

Also relevant for comparison: `bin/harden-sshd-keyonly`, `bin/cluster-status` (existing house style).

## What to look for (not exhaustive)
1. **fleet-audit correctness**: quoting and word-splitting; `~` expansion when the remote login shell
   is fish; unreachable hosts or ssh failures being reported as "clean"; repos with spaces or odd names;
   nested repos (e.g. a non-git dir that CONTAINS git repos, like `kirby/` containing `katam/.git`);
   worktrees; detached HEAD; submodules; branches with no upstream; how UNPUSHED is computed; whether
   gitignored data (the real risk on research repos) is visible at all; the "registered but not found"
   check.
2. **fleet-onboard safety**: `set -euo pipefail` interactions (`read -d ''`, `readlink` on a plain file,
   `tailscale status` exit codes); sudo / SUDO_USER / env preservation; a resolv.conf change that could
   break DNS; partial-failure states and reruns; the embedded heredoc (does the fish menu survive
   byte-for-byte, including `$c` and quotes?); file ownership in the user's home.
3. **Docs vs behaviour**: anything in README/hosts.yaml comments that the scripts don't actually do.
4. **Security**: TS_AUTHKEY handling (argv exposure, logs), anything written world-readable that
   shouldn't be.

## Deliverable
Write `reviews/4e91f9d/review_<model-slug>.md` containing:
- Findings ranked most severe first. Each one gives the file:line, what goes wrong, a CONCRETE input or
  state that triggers it, and a suggested fix.
- Severity tags: BLOCKER (data loss / lockout / audit says safe when not) · MAJOR · MINOR · NIT.
- What you checked and found correct. That matters too.
- A bottom line: is this safe to merge into main and use to onboard the 3 machines? If not, what
  must change first?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Inlined diff of the modified docs (main...fleet)
```diff
diff --git a/AGENT.md b/AGENT.md
index 77919fd..848d5f7 100644
--- a/AGENT.md
+++ b/AGENT.md
@@ -24,6 +24,8 @@
 
 When `ssh -o BatchMode=yes <host>` fails or a job vanishes, run **`cluster-status [host …]`** (`bin/`) FIRST — it probes each host's master + live reachability so you can tell *your* auth expired (→ `cluster-login <host>`) from *the cluster* being down/slow, and prints the Alliance status URL (https://status.alliancecan.ca/, JS-rendered, no stable JSON API) for incidents/maintenance. e.g. it distinguishes a working `trillium` login node from a down `trillium-gpu` one.
 
+For my own machines (laptop + lab workstations over Tailscale), see **`fleet/README.md`**. `fleet/hosts.yaml` is the inventory and `fleet/projects.yaml` records which host owns each repo. **`fleet-audit`** (`bin/`) lists every repo on every host and flags unsaved work (no remote, unpushed, dirty) and copies on the wrong host. **`fleet-onboard`** (`bin/`) adds a new CachyOS machine to the tailnet.
+
 To check storage, run **`cluster-quota [host …]`** (`bin/`). It runs `diskusage_report` on every cluster in parallel (default: fir, narval, rorqual, nibi, trillium, plus the PAICE clusters tamia, killarney and vulcan for aip-kjerbi) and prints one table of space **and** file-count usage per filesystem and allocation. Anything at ≥80% is flagged, and anything over quota is flagged OVER. A second table shows how much of each shared project/nearline allocation belongs to one user (`--user <name>`, default: you). It uses the cluster's nightly per-user breakdown where one exists (fir, narval). Elsewhere it does a live, niced metadata scan of `/project|/nearline/<alloc>/<user>`, which misses anything the user keeps outside that directory. `--only rrg-kjerbi` narrows it to one allocation, and `--no-scan` skips the scan. It's read-only and light enough for a login node, taking about 40 s for all clusters.
 
 **Recommended — symlink it into a dir already on your PATH** (e.g. `~/.local/bin`):
diff --git a/lab-access.md b/lab-access.md
index 17d9e8f..28225f1 100644
--- a/lab-access.md
+++ b/lab-access.md
@@ -19,8 +19,11 @@ later `ssh`/`rsync`/`scp` to that host reuses it — no re-auth, instant reconne
 
 ## The lab workstation — LKJ008-psy
 
-Tailscale host, **password auth**: the SSH key was removed so a second person can reach it via a
-Tailscale share (keys were causing trouble there), so no `ssh-copy-id` / keyless login. Alias:
+Tailscale host, **key + password auth**. The laptop's `id_ed25519` is in `~/.ssh/authorized_keys`
+again, so login needs no password (verified 2026-10-23). Password login is still enabled server-side
+on purpose, for `ssh-copy-id` and LAN bootstrap. (The key was briefly removed while the box was
+shared with a second person over Tailscale; that sharing has ended.) The box is in the fleet inventory,
+`fleet/hosts.yaml`. To add more lab machines, see `fleet/README.md`. Alias:
 
 ```sshconfig
 Host lab lkj008 lkj008-psy
@@ -29,8 +32,9 @@ Host lab lkj008 lkj008-psy
   # inherits ControlMaster/ControlPersist from Host *
 ```
 
-First `ssh lab` prompts for the password and opens the multiplexed master; after that,
-`ssh -o BatchMode=yes lab` (and rsync/scp, and agents) reuse the socket with no re-prompt.
+The key logs in with no prompt, and the first `ssh lab` also opens the multiplexed master that later
+`ssh -o BatchMode=yes lab` calls (and rsync/scp, and agents) reuse. Because it's key auth, BatchMode
+calls succeed even when that socket is down.
 
 Then it behaves exactly like a cluster alias:
 
```
