# Fleet: my own machines over Tailscale

The Alliance clusters are covered in `AGENT.md`. This folder covers the machines I own or
command myself: the laptop and the lab workstations, all on the tailnet `tail2e782e.ts.net`.

| File | What it is |
|---|---|
| `hosts.yaml` | Inventory: every machine, its ssh alias, Tailscale name/IP, spec, role |
| `projects.yaml` | Placement: which host owns each project (one host, except shared infra) |
| `../bin/fleet-onboard` | Run **on a new machine** to put it on the tailnet with sshd |
| `../bin/fleet-audit` | Run **from the laptop**: every repo on every host, with anything unsaved or misplaced |
| `ONBOARDING-AGENT.md` | Handoff for an agent (or you) adding a lab machine: who does what, checks, troubleshooting |

## The rules

1. **One project, one host.** Research goes on the lab machines by default. The laptop keeps
   desktop/personal things. Only infra (`cluster-utilities`, `agent-utilities`,
   `system-utilities`) lives on every host.
2. **Repos live in `~/Projects`.** `~/code` is legacy and being emptied (fleet-audit flags it as LEGACY).
3. **Before deleting a copy**, all of these must hold:
   - a `fleet-audit --fetch --all` run ends with every host `audited` and exits 0. A host marked
     `NOT AUDITED` or `INCOMPLETE` proves nothing, because a missing row is not a clean row. (A
     `skipped host` line is fine only if that host isn't the one you're deleting from.)
   - the copy's row is **present** and shows `ok`, or only these info flags: BEHIND, MISPLACED,
     UNLISTED, LEGACY, EMPTY (an empty dir: `rmdir` it). `UNDECIDED` means decide the owner first.
   - **Stop flags.** Any of these means the copy holds something not saved elsewhere: NOGIT, NESTED,
     NOREMOTE, LOCALREMOTE, SUBMODULE-NOBACKUP, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, HIDDENIDX,
     LFS, LFS-EXTERNAL, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then
     re-run.
     `IGNORED` and `LFS` stay on re-runs, so they need a check you do yourself (and note down):
     - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
       Git can't vouch for it.
     - `LFS:n`: **keep the copy, or archive the whole repo directory first** with the recipe below,
       but only if the row has **no `LFS-EXTERNAL`**.
     - `LFS-EXTERNAL`: **keep the copy.** The LFS store, or part of it, lives outside the repo directory
       (an absolute `lfs.storage`, a symlinked `objects` dir or shards/objects inside it, or a linked
       worktree using the main repo's store), so an archive of the repo wouldn't contain it. Archiving the
       store separately doesn't help either: tar stores symlinks as links, and the checks still pass.
     - For either flag, a successful `git lfs push --all` is NOT enough: `lfs.url` can send the upload
       somewhere other than the forge, and it skips LFS objects no commit points to (e.g. a file staged
       and then reset). Both still exit 0. Don't empty the store to make a flag go away.
     - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
       (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
     - `REPLACE:n`: `git replace -l` lists the replaced objects. If a replacement holds content you
       want, commit that content normally. Then `git replace -d <sha>` (the replaced object's sha, as
       `git replace -l` prints it) and re-run.
   - Deleting a whole root (e.g. `~/code`) also needs no `LOOSE:n` row for it.

   **What counts as "saved elsewhere":** only a remote on a forge (github.com, gitlab.com,
   git.overleaf.com, bitbucket.org, codeberg.org), checked by its configured URL, its rewritten
   (insteadOf) URL and every push URL. A copy on another fleet box (`user@lab:…`, `ssh://lab/…`) or a local path is not a backup.

   **Not counted as rebuildable** (so they show up as IGNORED): `.ipynb_checkpoints`, `.cache`,
   `env/`. Only `.venv`, `venv`, `__pycache__`, `node_modules`, pytest/mypy/ruff caches, `.tox`/`.nox`
   and `*.egg-info` are skipped.

   **Out of scope**, so deleting a copy loses these:
   - reflog-only commits (work you already reset or amended away);
   - annotated tag objects you never pushed, when they point at a commit that *is* pushed;
   - git-annex content.

   If any of that matters for a repo, archive the **whole repo directory** before deleting it,
   `.git` included (that's where the reflog, LFS store and annex objects live). Each archive goes in its
   own new directory, so archiving `~/code/x` and `~/Projects/x` on the same day can't overwrite each
   other. **Bash syntax**: your login shell is fish, so run `bash` first.
   ```bash
   # 0. stop anything still writing to the repo (jobs, editors, agents). tar -d below catches files that
   #    CHANGE after the archive is made, but not files ADDED afterwards.
   cd <parent-of-repo>                       # not inside the repo: tar would read its own output
   mkdir -p ~/archives                       # outside every fleet root (or fleet-audit flags it LOOSE)
   d=$(mktemp -d ~/archives/"$(hostname)-<repo>-$(date +%F)-XXXXXX")
   printf 'host=%s\nsource=%s\n' "$(hostname)" "$PWD/<repo>" > "$d/SOURCE"
   tar -czf "$d/<repo>.tar.gz" <repo> && tar -dzf "$d/<repo>.tar.gz" \
     && (cd "$d" && sha256sum <repo>.tar.gz > SHA256SUMS) && echo "VERIFIED: $d"
   # copy that directory to ANOTHER machine:
   #  - only if the step above produced a real hash (otherwise two empty strings would "match")
   #  - reserve the name there with a plain mkdir, which fails if it already exists (the random suffix
   #    is only unique on THIS machine), so nothing there is ever overwritten
   #  - check the bytes that arrived against THIS machine's hash, not a copied SHA256SUMS
   b=$(basename "$d"); h=$(cut -d' ' -f1 "$d/SHA256SUMS" 2>/dev/null)
   [[ "$h" =~ ^[0-9a-f]{64}$ ]] \
     && ssh <other-host> "mkdir -p archives && mkdir archives/$b" \
     && rsync -a "$d/" "<other-host>:archives/$b/" \
     && [ "$(ssh <other-host> "sha256sum archives/$b/<repo>.tar.gz" | cut -d' ' -f1)" = "$h" ] \
     && echo "COPY OK: <other-host>:archives/$b" || echo "!! copy NOT verified: don't delete"
   ```
   `tar -d` compares the archive with the files on disk. Delete the repo only after **both**
   `VERIFIED` and `COPY OK`. The quoted remote commands also run under fish (`&&` needs fish 3+, which
   CachyOS has). If the copy step fails after the remote `mkdir` succeeded (e.g. rsync was
   interrupted), check `archives/<that dir>` on the other host, remove it, and run the copy step again.
   `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
   objects (LFS, annex).

## Moving gitignored data to the owner host

`rsync --ignore-existing` + "compare sizes" is NOT enough: it keeps an older, different file at the
destination, and equal sizes don't mean equal contents. Instead:
```bash
# 0. stop anything still writing to the data
# 1. copy into a separate staging dir, so nothing on the destination is overwritten
rsync -aP <repo>/data/ lab:Projects/<repo>/_incoming-from-laptop/data/
# 2. verify by checksum. Success = exit status 0 AND no output. Any listed file differs: re-run 1, then 2.
rsync -anci <repo>/data/ lab:Projects/<repo>/_incoming-from-laptop/data/; echo "exit=$?"
# 3. on the owner host, merge the staging dir by hand. Where both sides have a file and they differ,
#    decide which to keep; don't let a tool silently pick.
```
Until it's merged, `_incoming-from-laptop/` shows as UNTRACKED on the owner host. That's the reminder.
Only after step 2 passes is the source copy's data safe to delete.

## Adding a lab machine (CachyOS already installed)

**0. Optional: make an auth key** so the box needs no browser login. In
<https://login.tailscale.com/admin/settings/keys>, generate an auth key (reusable if you're doing
several machines, short expiry). Treat it like a password and don't commit it.

**1. On the new machine, at its keyboard.** Get the script, either by copying it from the
existing lab box over the LAN (sshd + password are on there):
```bash
scp yorguin@<lkj008-lan-ip>:Projects/cluster-utilities/bin/fleet-onboard .
```
or by cloning the repo if the machine has GitHub access. Then:
```bash
bash fleet-onboard lkj0XX-psy             # interactive: open the printed URL anywhere
bash fleet-onboard --authkey lkj0XX-psy   # or: prompts (hidden) for the key from step 0
```
Never put the key on the command line or in an env var you type: it would land in shell history and
the process list. `--authkey` reads it hidden and hands it to tailscale through a 0600 temp file.

It runs `pacman -Syu` (full upgrade, so openssh isn't half-upgraded), installs and enables `sshd` +
`tailscaled` (plus `bind`, for `dig`), applies the CachyOS MagicDNS fix (the resolv.conf symlink),
and joins the tailnet. If the upgrade installed a new kernel, tailscale's `tun` module can't load
until a reboot. It then says so and exits: reboot and run it again. The DNS change is guarded. It
only switches if DNS works beforehand and resolved's stub listener answers, and it keeps a timestamped
`/etc/resolv.conf.bak-*`. Afterwards it checks DNS through `/etc/resolv.conf` itself and restores the
backup (exit 1) if that broke. It refuses to run over SSH, which it detects even through sudo, because
the restarts can cut the session. `--force` overrides that. It leaves password login on. It also installs the SSH
login menu (1 plain fish / 2 tmux / 3 herdr, plain fish after 5 s) into `~/.config/fish/conf.d/`, but only
if your login shell is fish. That menu is an embedded copy of
`linux-config/home/dot_config/fish/conf.d/ssh-session-menu.fish`, so if you change the menu there,
re-paste it into the script. herdr itself isn't installed, so option 3 says so until you install it.

**1b. Stop it sleeping.** KDE suspends after 15 min idle by default, and a suspended box is off the
tailnet. Mask the sleep targets (covers the login screen and every user) and set powerdevil's idle
action to "do nothing" (so idle can't shut it down either):
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
for p in AC Battery LowBattery; do kwriteconfig6 --file powerdevilrc --group $p --group SuspendAndShutdown --key AutoSuspendAction 0; done
```

**1c. Clock.** The UdeM network blocks NTP (wired and Wi-Fi), so lab clocks drift by minutes to
hours. `fleet-onboard` runs `bin/fleet-clock-install`: `clock-from-https` (sets the clock from the
`Date:` header of a few big HTTPS sites, majority vote, steps only beyond 2 s, writes the RTC) plus an
hourly timer. It does nothing while NTP is synchronized, so off campus timesyncd stays in charge.
Check any box with `bin/clock-from-https --dry-run`; if UdeM IT ever names an NTP server, switch to a
timesyncd drop-in (`NTP=` then `NTP=<server>`, since CachyOS ships its own `NTP=`) and drop this.

**2. In the admin console**, disable key expiry for the new machine. Otherwise it silently
drops off the tailnet after ~180 days, which you'd only notice once you're remote.

**3. On the laptop**, using the snippet `fleet-onboard` printed:
```sshconfig
Host lab12 lkj012 lkj012-psy   # alias from the hostname: lkj0NN-psy -> labNN
  HostName 100.x.y.z        # Tailscale IP
  User yorguin
```
```bash
ssh-copy-id lab12                         # password once
ssh -o BatchMode=yes lab12 hostname       # must succeed with no prompt; that's what agents use
```
The global `Host *` ControlMaster block (`../lab-access.md`) applies automatically.

**4. Reboot the new machine once**, then from the laptop check that `ssh -o BatchMode=yes <alias>`
and `ssh lkj0XX-psy` (MagicDNS) still work. Do this before onboarding the next machine.

**5. Register it.** Add a stanza to `hosts.yaml` with `status: active`, then run `fleet-audit` to
confirm it's reachable (`== <id>: audited`). Infra repos are `[*]` in `projects.yaml`, so they're
expected on the new host with no edit needed.

**6. Lab ↔ lab mesh.** `bin/fleet-mesh` (on the laptop) makes every active lab box in `hosts.yaml`
trust every other one: each gets a key, the others' keys in `authorized_keys` (commented
`fleet-mesh:<name>`, so one box can be revoked), their host keys in `known_hosts` (read over the
laptop's verified connections, not keyscanned), and the same `labNN` aliases in a marked block of
`~/.ssh/config`. It finishes with an every-pair check. Idempotent; `--check` only checks. The laptop
reaches in; the lab boxes don't get a key to the laptop.

## Auditing

```bash
fleet-audit                 # problems only, every active host (~5 s)
fleet-audit --fetch --all   # the run to use before deleting anything (fetch --prune, every row)
fleet-audit --size          # add du sizes, incl. each repo's ignored data (slow on the big data dirs)
fleet-audit lab8            # one host
FLEET_AUDIT_TIMEOUT=3600 fleet-audit --fetch --size   # raise the per-host time limit
```
`fleet-audit --help` explains every flag. Exit 0 means every requested host was fully audited, 1 means
some host was not, and 2 means a usage or inventory error.

`UNPUSHED` counts commits reachable from any local ref (branches, tags, notes, replace refs, every
worktree's HEAD; not the stash) that the forge remotes don't have. With `--fetch`, "have" means the
server advertises it right now (`git ls-remote`). A branch that was squash-merged and then deleted on
GitHub shows up here too. To see the actual commits before assuming work is at risk, ask the
server the same way the tool does (forge remote `origin` here). Don't use `--not --remotes=origin`,
because cached tracking refs can be stale:
```bash
# bash syntax (in fish: run it inside `bash -c '…'`). The && chain stops if the server can't be reached;
# otherwise an empty answer would make git log list ALL local history as unpushed.
git fetch --prune origin && adv=$(git ls-remote origin) && printf '%s\n' "$adv" \
  | awk -F'\t' 'NF {print $1}' | git cat-file --batch-check='%(objectname) %(objecttype)' \
  | awk '$2 ~ /^(commit|tag|tree|blob)$/ {print "^" $1}' \
  | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin \
  || echo "!! couldn't ask origin (fetch or ls-remote failed): no answer, NOT 'nothing unpushed'"
```

Keep `hosts.yaml` / `projects.yaml` in their flat format, because they are read with awk. Inline
`# comments` are fine. In `hosts.yaml`, any line the parser doesn't understand stops the run with
exit 2 before anything is scanned. That covers a tab in the indentation or content (trailing whitespace
and comments are stripped first), a host id at column 0, a stray top-level key, a missing status, and a
root that doesn't start with `/` or `~`. Roots are space-separated, so a path with spaces isn't
supported. In `projects.yaml` it's a warning.

## Auth notes

- **Keys + password both work** on the lab boxes. Password login was kept on purpose
  (`ssh-copy-id`, LAN bootstrap). If a box should ever be key-only, `../bin/harden-sshd-keyonly`
  does that safely.
- Agents always use `ssh -o BatchMode=yes <alias>`. With a key, that works even if the
  ControlMaster socket is down.
