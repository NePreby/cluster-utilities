# Fleet: my own machines over Tailscale

The Alliance clusters are covered in `AGENT.md`. This folder covers the machines I own or
command myself: the laptop and the lab workstations, all on the tailnet `tail2e782e.ts.net`.

| File | What it is |
|---|---|
| `hosts.yaml` | Inventory: every machine, its ssh alias, Tailscale name/IP, spec, role |
| `projects.yaml` | Placement: which host owns each project (one host, except shared infra) |
| `../bin/fleet-onboard` | Run **on a new machine** to put it on the tailnet with sshd |
| `../bin/fleet-audit` | Run **from the laptop**: every repo on every host, with anything unsaved or misplaced |

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
     LFS, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run.
     `IGNORED` and `LFS` stay on re-runs even when resolved, so for those two, a check you do yourself
     (and note down) is what clears them:
     - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
       Git can't vouch for it.
     - `LFS:n`: the local LFS store stays after a successful upload, so the flag stays too. Run
       `git lfs push --all <forge-remote>` (it uploads only what the server lacks). If that exits 0,
       the LFS content is safe and this flag no longer blocks deleting *that* copy. Don't empty the
       store just to make the flag go away.
     - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
       (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
     - `REPLACE:n`: `git for-each-ref refs/replace` lists them. If a replacement holds content you
       want, commit that content normally. Then `git replace -d <ref>` and re-run.
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
   `.git` included (that's where the reflog, LFS store and annex objects live), and check the archive:
   ```bash
   tar -C <parent> -czf <repo>-$(date +%F).tar.gz <repo> && tar -tzf <repo>-$(date +%F).tar.gz >/dev/null && echo OK
   ```
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

**2. In the admin console**, disable key expiry for the new machine. Otherwise it silently
drops off the tailnet after ~180 days, which you'd only notice once you're remote.

**3. On the laptop**, using the snippet `fleet-onboard` printed:
```sshconfig
Host lab2 lkj0XX-psy
  HostName 100.x.y.z        # Tailscale IP
  User yorguin
```
```bash
ssh-copy-id lab2                          # password once
ssh -o BatchMode=yes lab2 hostname        # must succeed with no prompt; that's what agents use
```
The global `Host *` ControlMaster block (`../lab-access.md`) applies automatically.

**4. Reboot the new machine once**, then from the laptop check that `ssh -o BatchMode=yes <alias>`
and `ssh lkj0XX-psy` (MagicDNS) still work. Do this before onboarding the next machine.

**5. Register it.** Add a stanza to `hosts.yaml` with `status: active`, then run `fleet-audit` to
confirm it's reachable (`== <id>: audited`). Infra repos are `[*]` in `projects.yaml`, so they're
expected on the new host with no edit needed.

## Auditing

```bash
fleet-audit                 # problems only, every active host (~5 s)
fleet-audit --fetch --all   # the run to use before deleting anything (fetch --prune, every row)
fleet-audit --size          # add du sizes, incl. each repo's ignored data (slow on the big data dirs)
fleet-audit lab             # one host
FLEET_AUDIT_TIMEOUT=3600 fleet-audit --fetch --size   # raise the per-host time limit
```
`fleet-audit --help` explains every flag. Exit 0 means every requested host was fully audited, 1 means
some host was not, and 2 means a usage or inventory error.

`UNPUSHED` counts commits reachable from any local ref (branches, tags, notes, replace refs, every
worktree's HEAD; not the stash) that the forge remotes don't have. With `--fetch`, "have" means the
server advertises it right now (`git ls-remote`). A branch that was squash-merged and then deleted on
GitHub shows up here too. To see the actual commits before assuming work is at risk (forge remote
`origin` here):
```bash
git fetch --prune origin
git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --not --remotes=origin
```

Keep `hosts.yaml` / `projects.yaml` in their flat format, because they are read with awk. Inline
`# comments` are fine. In `hosts.yaml`, any line the parser doesn't understand (a tab, a host id at
column 0, a stray top-level key, a missing status) stops the run with exit 2 before anything is
scanned. In `projects.yaml` it's a warning.

## Auth notes

- **Keys + password both work** on the lab boxes. Password login was kept on purpose
  (`ssh-copy-id`, LAN bootstrap). If a box should ever be key-only, `../bin/harden-sshd-keyonly`
  does that safely.
- Agents always use `ssh -o BatchMode=yes <alias>`. With a key, that works even if the
  ControlMaster socket is down.
