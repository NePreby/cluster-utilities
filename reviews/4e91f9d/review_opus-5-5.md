# Review of `fleet` @ 4e91f9d by Claude Opus 5.5 (`claude-opus-5-5`)

Scope: `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/{hosts,projects}.yaml`, `fleet/README.md`, the
AGENT.md / lab-access.md diff, compared against `bin/harden-sshd-keyonly` and `bin/cluster-status`.
I read every file in full. I did not read anything else under `reviews/`.

**How I tested.** I copied the SCAN heredoc out of `fleet-audit` and ran it (git 2.56, bash 5.3)
against a scratch `$HOME` that had a bare "GitHub" remote and one repo per edge case. I then ran the
full `fleet-audit` from a copy of the repo with modified `hosts.yaml` files. For the remote path I put
a fake `ssh` on PATH that runs `fish -c "<joined argv>"`, which is what sshd does when the login shell
is fish. For `fleet-onboard` I took the `read -d '' MENU` block, ran it under `set -euo pipefail`, and
`cmp`'d the result with the heredoc body. I also tested `install -d` on an existing 0700 directory. I
could not test tailscale, NetworkManager, resolved or pacman, so what I say about those comes from
reading the code. Results marked **[verified]** were reproduced. The rest are from reading the code.

---

## Findings, most severe first

### BLOCKER 1: commits on a detached HEAD are invisible, and DETACHED does not count as a flag [verified]
`bin/fleet-audit:75` and `:123`. `UNPUSHED` comes from `git log --branches --not --remotes`, and
`--branches` does not include `HEAD`. Commits made on a detached HEAD are therefore never counted. The
scanner sets `br=DETACHED` but never adds a flag for it, so in default mode such a row has no flags
and is hidden (line 123).
*Trigger:* `git checkout --detach && git commit …`. This is common after `git rebase` stops partway,
a `git bisect`, or an agent checking out a SHA. My `detached` test repo came back as
`DETACHED 0 0 0 0 0`. If its name were registered for that host the row would not print at all, and
deleting the copy loses the commit.
*Fix:* use `git log --branches --tags HEAD --not --remotes` (HEAD added explicitly). Make `DETACHED` a
flag whenever HEAD is not reachable from any remote-tracking ref.

### BLOCKER 2: unpushed commits inside submodules are invisible [verified]
`bin/fleet-audit:73-75`. The superproject's `status` and `log` don't look inside the submodule's
refs. *Trigger:* commit in `sm/`, `git add sm && git commit && git push` in the parent, but never push
inside `sm/`. My `withsub` repo reported all zeros while `sm/` was `[ahead 1]`. After the copy is
deleted, the parent repo on GitHub points to a submodule SHA that exists nowhere.
*Fix:* run `git submodule foreach --recursive 'git log --branches HEAD --not --remotes --oneline'`
and add its output to `unp`, or add a `SUBMODULE` flag and require a manual check.

### BLOCKER 3: uncommitted work in linked worktrees is invisible [verified]
`bin/fleet-audit:73-74`. `git status` only looks at the main worktree. A worktree placed inside the
repo and gitignored, such as `.worktrees/feat` or the `.claude/worktrees/*` that agent tools create,
holds uncommitted files that show up in neither DIRTY nor UNTRACKED. `rm -rf` on the copy deletes
them too. My `wt` repo, with `wip.txt` in a gitignored in-repo worktree, came back all zeros. (Commits
on the worktree's branch are counted, because refs are shared. Only uncommitted work is lost.)
*Fix:* loop over `git worktree list --porcelain` and run the dirty and untracked checks in each
worktree. Add a `WORKTREES:n` flag when n > 1.

### MAJOR 4: "pushed" is checked against stale local remote-tracking refs, and any remote counts
`bin/fleet-audit:71,75`. `--not --remotes` trusts `refs/remotes/*` exactly as it is on disk. Nothing
ever prunes it: `--fetch` runs `fetch --all` without `--prune`, and fetch errors go to `2>/dev/null`.
**[verified]** In my `stale` repo, branch `gone` was pushed and then deleted on the remote without a
prune. The commits are still counted as pushed and the repo reports clean. The same thing happens when
the GitHub repo has been deleted or made inaccessible. Separately, any remote counts as "saved",
including a path remote such as `origin = /home/yorguin/code/foo` (common after cloning `~/code/foo`
into `~/Projects`) or another fleet host (`lab:Projects/foo`). Only the first remote's URL is shown
(line 69), so you can't see any of this in the output.
*Fix:* use `fetch --all --prune` and add a `FETCHFAIL` flag when it fails. Add a `LOCALREMOTE` flag
for remotes that aren't `github.com`. Document that UNPUSHED is only meaningful after `--fetch`.

### MAJOR 5: loose files and dot-dirs in a root are never listed [verified]
`bin/fleet-audit:62`. `"$r"/*/` matches only non-hidden directories. In my test,
`~/Projects/notes.txt` and `~/Projects/.hidden/` did not appear anywhere. README rule 2 says
`~/code` "is being emptied". The natural last step is `rm -rf ~/code` once the audit is clean, and
that silently deletes any loose `analysis.py`, `data.zip` or `.ipynb_checkpoints`.
*Fix:* add one row per root, for example `LOOSE:n`, that counts entries other than non-hidden
directories (`find "$r" -mindepth 1 -maxdepth 1 \( -name '.*' -o ! -type d \)`).

### MAJOR 6: gitignored data gives no signal at all
`bin/fleet-audit:73-74`. This is documented in README rule 3, but your own `projects.yaml:73` says
the real payload is "67 GB of ignored data/derivatives". The audit prints exactly the same thing for
an empty clone and for a clone holding 67 GB of only-copy results. **[verified]** My `ignored` repo,
with `data/` excluded, came back all zeros. For research repos this is where the main data-loss risk
is, and the tool says nothing about it.
*Fix:* count ignored paths cheaply with `git status --porcelain --ignored=matching | grep -c '^!!'`
and show an `IGNORED:n` flag. Optionally run `du` on them with `--size`. You can exclude obvious
regenerables (`.venv`, `__pycache__`, `node_modules`) if that's too noisy.

### MAJOR 7: a failed or partial scan exits 0 and looks like a clean report [verified]
`bin/fleet-audit:96,105-107,129-136`. When ssh fails, the only signal is one `!!` line on **stderr**,
and the exit status is 0. In default mode a missing row already means "clean and correctly placed",
so an unreached host and a clean host look the same on stdout. I ran it with lab unreachable and
stdout redirected to a file (`fleet-audit lab > out.txt`): `out.txt` held only the header plus about
30 bogus "registered for lab but not found there" lines, and the exit code was 0. An agent that checks
`$?`, or someone reading a saved report, sees a pass.
Partial scans are worse. `timeout 600` (line 95) kills a slow scan partway, which is likely with
`--size`/`--fetch` over about 30 lab repos with 60 s fetch timeouts. Every repo after that point is
missing, and the output looks the same as when they are clean.
The "not found" check (line 133) also runs against the empty `.tsv` of a failed host.
*Fix:* have the scanner print a final `#DONE` sentinel line. Treat a missing sentinel or a non-zero
status as failure. Print `!! HOST lab NOT AUDITED` on stdout, skip the not-found check for that host,
and `exit 1`. The README should say to confirm a row exists with `--all` before deleting anything.

### MAJOR 8: small hosts.yaml mistakes silently remove hosts from the audit [verified]
`bin/fleet-audit:42-46,84-86`.
- **An inline comment on a host-id line** (`  lab2:   # new box`) fails the `:[[:space:]]*$` regex.
  The stanza then merges into the previous host, and its `ssh/roots/status` overwrite that host's
  values. In my test the real `lab` (ssh `nosuchhost`) was **never contacted and no failure was
  printed**, while lab2's dirs were reported as host "lab". This is exactly the edit someone makes when
  uncommenting the lab2 template.
- **Trailing whitespace** after `status: active ` (no comment) is not stripped, because the
  `sub(/[[:space:]]+#.*/…)` only removes whitespace that is followed by `#`. The host is dropped with no
  message. The same applies to `ssh: local ` and to `status: Active`/`"active"`.
- **An unknown host argument**: `fleet-audit lba` prints only the header and exits 0, which looks like
  an all-clean report.
*Fix:* strip trailing whitespace and inline comments on every line, including id lines, before
matching. Print every non-`active` host it skipped, with the reason. Exit 2 when a requested id
doesn't match an active host.

### MAJOR 9 (onboard): the resolv.conf rewrite has no check that DNS still works and no rollback
`bin/fleet-onboard:89-98`. The script repoints `/etc/resolv.conf` and restarts resolved,
NetworkManager and tailscaled without checking that `$stub` exists or that DNS still works afterwards.
On a default CachyOS install NM picks up the symlink and pushes DNS to resolved, so this is the fix
Tailscale documents. On a customised lab box it breaks name resolution: `DNSStubListener=no` (stub
points at a dead 127.0.0.53), NM `dns=none`/`rc-manager=unmanaged` with a hand-written campus
`resolv.conf` (resolved falls back to public FallbackDNS, which campuses often block), or dnsmasq. The
backup is created, but nothing prints its name as the way to recover, and `cp … || true` (line 93)
still continues if the backup itself failed. This is unlikely, but the brief requires that the script
never cut the machine off the network.
*Fix:* refuse unless `[ -e "$stub" ]`. After the restarts, run
`getent hosts archlinux.org || { restore backup; restart NM; exit 1; }`. Print the backup path.

### MINOR 10: commits reachable only from tags are missed
`bin/fleet-audit:75`. My `tagonly` repo had a commit on a detached HEAD, then `git tag v1`, then
switched back: all zeros. *Fix:* add `--tags`, as in finding 1.

### MINOR 11: `status.showUntrackedFiles=no` hides untracked files [verified]
`bin/fleet-audit:74`. When the repo config sets this, `git status --porcelain` lists no `??` lines.
My `untrk_cfg` repo, holding a new file, reported `UNTRACKED:0`. *Fix:* pass `-unormal` explicitly.

### MINOR 12: every git failure counts as zero
`bin/fleet-audit:73-77`. `2>/dev/null | wc -l`, `|| echo 0`, `2>/dev/null` on fetch. If a repo is
corrupt or a command errors, the result is "0 problems". (Dubious ownership is handled
conservatively, since `rev-parse` fails and the repo is reported as NOGIT.) *Fix:* record a failure
and print an `ERR` flag. Don't default to 0.

### MINOR 13: the audit is not read-only, and `--fetch` can stall on a prompt
`bin/fleet-audit:6,71,73-74`. `git status` refreshes the index and takes `index.lock`, which can make
a git command that an agent or you are running on that host fail at the same moment. Use
`git --no-optional-locks`. On the laptop, `--fetch` against `git@github.com` remotes with a key that
has a passphrase and no running agent prompts on the tty from a background job. `GIT_TERMINAL_PROMPT`
only covers HTTPS. *Fix:* set `GIT_SSH_COMMAND='ssh -o BatchMode=yes'`.

### MINOR 14: an empty or missing-but-globbed root prints a bogus `*` NOGIT row [verified]
`bin/fleet-audit:62`. With no matching dirs the glob stays literal, so the output gets `* … NOGIT
UNLISTED`. *Fix:* `shopt -s nullglob` in SCAN. Use `dotglob` too, or list dot-dirs separately as in
finding 5.

### MINOR 15 (onboard): `install -d` changes an existing `~/.config` to mode 755 [verified]
`bin/fleet-onboard:83`. GNU `install -d` applies its mode, 0755 by default, to directories that
already exist. I tested it: a 0700 dir became 755. On a shared lab box this exposes the names of
everything in `~/.config`, such as `gh/` and `rclone/`. *Fix:* create each missing component with
`[ -d x ] || install -d …`, or `sudo -u "$user" mkdir -p`.

### MINOR 16 (onboard): the auth key ends up in argv and in fish history
`bin/fleet-onboard:5,112`, `README.md:140`. `tailscale up --auth-key="$TS_AUTHKEY"` puts the key in
`/proc/*/cmdline`, where every local user can read it while `up` blocks. The documented invocation
`TS_AUTHKEY=tskey-… ./fleet-onboard` also lands in `~/.local/share/fish/fish_history`. The README
suggests a *reusable* key for several machines, so that key stays live until it expires.
*Fix:* `read -rs` the key from a prompt, or accept a key file, and pass `--auth-key=file:/path` with
mode 600, deleting the file afterwards. Tell users to start the command with a space so fish leaves it
out of history, or to clear it.

### MINOR 17 (onboard): `pacman -S` without `-yu`
`bin/fleet-onboard:37`. On a box that has been sitting since install, the sync DB is stale. That
gives 404s and the script aborts, which is at least safe. If the DB is fresher than the installed
system, you get a partial upgrade (e.g. a new `openssh` against an older `openssl`). That is the
Arch-unsupported case, and here it hits the package you need for remote access. *Fix:* use
`pacman -Syu --needed --noconfirm …`, or tell users to run `pacman -Syu` first.

### MINOR 18 (onboard): a rerun after the tailscaled restart can abort
`bin/fleet-onboard:97-107`. Three seconds after `restart tailscaled`, `tailscale status` can still
fail on an already-logged-in node. The script then runs a bare `tailscale up`, and tailscale rejects
that if the node had non-default flags (the "requires mentioning all non-default flags" error).
`set -e` then aborts after the network changes have been made. A rerun recovers, so this is minor.
*Fix:* wait until `tailscale status --json` reports a `BackendState`, and branch on `NeedsLogin`
rather than on the exit code.

### MINOR 19 (onboard): running it over SSH cuts its own session
`bin/fleet-onboard:96-97`. Restarting NM, and especially tailscaled, drops a Tailscale-SSH session
and leaves the script killed partway. The README says "at its keyboard", but the script's claim to be
idempotent invites running it on `lab` remotely just to install the menu. *Fix:* if `SSH_CONNECTION`
is set, warn, or skip the restarts unless `--force` is passed.

### MINOR 20 (docs/data): infra repos will show MISPLACED on every new host
`fleet/projects.yaml:60-62` lists only `[laptop, lab]`, but README rule 1 says infra lives on
"every host" and `fleet-onboard`'s hand-off says to clone `cluster-utilities` and `agent-utilities`
there. As soon as `lab2` becomes active, those copies show `MISPLACED(owner:laptop,lab)`. *Fix:* add
"append the host id to the infra lines" to README step 4, or support a `[*]` wildcard.

### NIT
- `fleet-audit:33`: `sed -n '2,24p'` also prints `set -uo pipefail` into `--help`. It should be `2,22p`.
- `fleet-audit:122`: LEGACY only matches the literal string `~/code`, not `/home/yorguin/code` or `~/code/`.
- `fleet-audit:117`: a key duplicated in projects.yaml makes `want` contain a newline, which breaks
  MISPLACED. A quoted key (`"foo": [lab]`) keeps its quotes. The awk could warn about both.
- `fleet-audit:124`: `${n:0:38}` can make two long names look identical.
- `fleet-audit` local branch (line 90) has no `timeout`, but the remote branch has 600 s.
- `fleet-onboard:23`: `exec sudo -- "$0"` fails when the script is started as `bash fleet-onboard`
  or has lost its exec bit (e.g. a browser download of the raw file). `harden-sshd-keyonly` has the
  same pattern. Use `exec sudo … bash "$(readlink -f "$0")" "$@"`.
- `fleet-onboard:84`: as root, `printf > "$dest"` follows a symlinked `dest` (stow or chezmoi symlink
  mode) and overwrites the source of truth in `linux-config` with the embedded copy.
- `fleet-onboard:25`: if run from a root login, `user=root` and the hand-off prints `User root`.
- The script doesn't check for an active firewall (ufw/firewalld). If one is on, `ssh lab2` fails
  after onboarding with no hint why. Check it once on the first new box.
- `lab-access.md`: "BatchMode calls succeed even when that socket is down" only holds when the key has
  no passphrase or is loaded in an agent the caller can reach.
- `README.md:135` scp's `Projects/cluster-utilities/…` from lkj008. Check that the repo is actually
  there and not still in `~/code`. `harden-sshd-keyonly:9` still uses the `~/code` path.

---

## Checked and found correct
- **`~` through fish:** `qroots` single-quotes each root, fish passes `'~/Projects'` to bash
  literally, and `${root/#\~/$HOME}` expands it on the remote. **[verified]** with the fish-emulating
  fake ssh. LEGACY then matches because roots are reported exactly as written.
- `bash -s` from a here-string: the whole scanner is one `for` compound, so bash reads all of it before
  running anything. A git child can't swallow the rest of the script from stdin.
- hosts.yaml: full-line comments, the commented-out `lab2` stanza (uncommenting `# ` gives the right
  indentation), and inline `# …` on value lines all parse correctly. projects.yaml: `[?]` becomes
  `?`, `[lab, laptop]` becomes `lab,laptop`, inline comments are stripped, commented keys are skipped,
  and all 38 entries parse. **[verified]**
- A non-git dir containing repos (`kirby/katam/.git`) gives NOGIT, which is conservative.
  **[verified]** NOGIT also covers bare repos, a `$HOME`-is-a-repo setup, and dubious ownership.
- A remote added but never fetched counts every commit as UNPUSHED, which is conservative.
  **[verified]** No remote gives NOREMOTE, no upstream gives BEHIND 0, and names with spaces survive
  the TSV.
- `read -r -d '' SCAN`: `fleet-audit` doesn't use `-e`, so read's status of 1 is harmless.
  `fleet-onboard:43` uses `|| true`.
- **Embedded fish menu:** after `read -d ''` and `printf '%s\n'` it is **byte-identical** to the
  heredoc body, `$c` and the quotes come through intact, `fish -n` passes, and the `$(cat)` equality
  makes reruns no-ops. **[verified]** The guards (`is-interactive`, `SSH_CONNECTION`, `TMUX`,
  `NO_TMUX`) keep it from firing for `fleet-audit`, scp/rsync or ssh-copy-id. Backups are named
  `*.fish.bak-DATE`, which fish's conf.d doesn't source.
- `set -e` traps: `readlink` on a plain file sits inside `[ ]` within `if`, and inside `|| echo`, so
  neither aborts. `is-active -q NetworkManager && restart` is safe. The `tailscale status | grep ||
  echo` hand-off is safe under pipefail.
- sudo: `--preserve-env=TS_AUTHKEY` works with the stock `%wheel ALL=(ALL:ALL) ALL`, since `ALL`
  implies SETENV. Home comes from `getent`, not root's `$HOME`. The menu file is chowned to the user
  with mode 644. The script never touches sshd auth config, so it can't lock you out over SSH.
- Docs agree with behaviour on: what onboard does and in what order, password login left on, the menu
  only installing for fish users, herdr not being installed, and the key-expiry hand-off.

---

## Bottom line

**`fleet-onboard`: OK to use on the 3 machines**, with two small changes made first or done by hand:
(a) guard the resolv.conf change with `[ -e "$stub" ]` plus a DNS check that rolls back on failure
(#9), and (b) don't put a reusable auth key on the command line. Use the interactive URL login or a
key file (#16). Running `pacman -Syu` first (#17) is a good idea. It is idempotent in the cases I could
check, and it can't lock you out over SSH.

**`fleet-audit`: do not merge it as a "safe to delete" oracle yet.** It reports clean on detached-HEAD
commits, unpushed submodule commits, dirty linked worktrees, stale remote-tracking refs and tag-only
commits (#1–4, #10), all reproduced. It never lists loose files or ignored data (#5–6). It also exits 0
with no rows when a host wasn't reached or a hosts.yaml line was slightly off (#7–8). Before anyone
deletes a copy on the strength of its output, fix these, in priority order:
1. Add `HEAD --tags` to the unpushed check, add a DETACHED flag, recurse into submodules, and check
   every worktree (#1, #2, #3, #10).
2. Add a completion sentinel per host. A host that failed or was not reached should print on stdout
   and make the script exit non-zero (#7). Reject hosts.yaml typos and unknown host ids loudly (#8).
3. Add `--prune` and a FETCHFAIL flag, and flag non-GitHub remotes (#4).
4. Add a LOOSE row per root and an IGNORED:n flag per repo (#5, #6), plus `-unormal` and `nullglob`
   (#11, #14).
5. README: deletion requires a fresh `fleet-audit --fetch --all` in which the row is **present** with
   `ok`, and no `!!` lines.

Until then, use it as a placement and inventory report only. Before deleting any copy, check it by
hand: `git status --ignored`, `git worktree list`, `git submodule status`,
`git log --branches --tags HEAD --not --remotes` after `git fetch --prune`, and `ls -A` on the root.
