# Round-3 review of `fleet` at 980b7c9 — Claude Fable 5.1 (`claude-fable-5-1`)

Scope: the diff 7a25a69..980b7c9 in `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/README.md`, read
against the full current files plus `fleet/hosts.yaml` and `fleet/projects.yaml`. I did not read anything
else under `reviews/`.

Method: I extracted the remote scanner heredoc from `bin/fleet-audit` and ran it against ~30 synthetic
repos (gone remotes, never-fetched forge remotes, fork setups, Overleaf URLs, insteadOf/pushInsteadOf,
glob remote names, submodules with lab-only remotes, LFS, assume-unchanged/skip-worktree, renames,
unreadable dirs, sparse checkouts, detached worktrees, notes, stashes), ran `--fetch` through a fake ssh
wrapper (`FLEET_AUDIT_GIT_SSH`) to exercise FETCHFAIL/prune/skipFetchAll, and ran the real `bin/fleet-audit
--all` end to end with a `ssh: local` host pointing at those roots. For `fleet-onboard` I probed this
CachyOS box (kernel 7.2.8-1-cachyos, OpenSSH 10.5, bind 9.20): `/sys/module/tun`, `CONFIG_TUN`,
`/proc/<pid>/stat` parsing of a real process chain, `dig` via `/etc/resolv.conf` and `@127.0.0.53`
100× under `pipefail`, `realpath -m`, nsswitch.

## Round-2 items

| Item | Verdict | What I tried |
|---|---|---|
| A1 strict hosts.yaml | **PARTIAL** | Quoted status, uppercase key, empty `roots:`, duplicate id, missing status → all exit 2 ✔. But a host id dedented to column 0 (`lab:`), a tab-indented stanza, or any stray top-level line mid-file **silently drops every host after it, exit 0** (see B2). |
| A2 `\x1f` rows, `-` for empty | FIXED | Rows with empty branch/url/size/flags parse into the right columns; `flags=()` when `-`; names with `\x1f` → BADNAME. |
| A3 ROOTERR / NOACCESS / find ERR | **PARTIAL** | Unreadable root → `INCOMPLETE … could not be listed`, exit 1 ✔. `chmod 000` entry → `ERR NOACCESS` ✔. Unreadable subdir inside a NOGIT dir → `NESTED:1 ERR` ✔. But an unreadable subdir **inside a git repo** holding untracked files → row shows **clean** (see B1). |
| A4 forge allow-list | FIXED | `user@lab:…`, `ssh://lab/…`, local path, insteadOf→lab → LOCALREMOTE ✔; `https://git@git.overleaf.com/…`, `ssh://git@GitHub.com:443/…`, `git@github.com:` → backup ✔; pushurl-only remote → `LOCALREMOTE ERR` (safe). `pushInsteadOf` is not checked (N5). |
| A5 orphan refs / skipFetchAll | FIXED | Commit covered only by `refs/remotes/gone/main` → UNPUSHED ✔. `skipFetchAll` remote fetched by name and pruned → deleted-on-server branch becomes UNPUSHED ✔. Stale `origin/main` reset by `--fetch` → UNPUSHED ✔. Auth failure → FETCHFAIL ✔. |
| A6 notes / replace / `--all` | FIXED | `git notes add` → UNPUSHED:1 ✔; detached HEAD in a linked worktree with a local-only commit → UNPUSHED:1 WORKTREES:1 ✔ (`--all` does include other worktrees' HEADs); `refs/stash` excluded, counted as STASH ✔. |
| A7 HIDDENIDX | FIXED (one spurious ERR, N1) | assume-unchanged + skip-worktree edited files → HIDDENIDX:2 ✔; skip-worktree file absent on disk → not counted ✔ but produces ERR when it is the last index entry. |
| A8 LFS | FIXED | Repo with one LFS object → LFS:1 ✔. |
| A9 REGEN | FIXED | `.ipynb_checkpoints/`, `env/` → IGNORED:2; `.venv/` skipped ✔. README matches. |
| `status -z` renames | FIXED | `git mv a "b c"` + untracked `d e` → DIRTY:1 UNTRACKED:1 ✔ (source record skipped, spaces kept). |
| O1 under_ssh pre/post sudo | FIXED | Chain walk via `sed 's/.*) //' | cut -f2` correct on a real 15-deep process chain; `comm` names `sshd-session`/`sshd-auth` exist on OpenSSH 10.5. Check runs before `exec sudo` and again after. |
| O-r1 tailscale status retry | FIXED | `|| true` inside `$( )` with pipefail; empty state after 20 s → clear exit 1. |
| O-r2 tun | FIXED (caveat N7) | `CONFIG_TUN=m` and `/sys/module/tun` present when loaded on linux-cachyos. |
| O-r3 dig | FIXED | `bind` owns `/usr/bin/dig`; `dig` via `/etc/resolv.conf` and `dig @127.0.0.53` both NOERROR; 0/100 SIGPIPE failures with `grep -q` under pipefail. `getent` goes through `resolve [!UNAVAIL=return]` as the comment says. |
| O-r4 NeedsMachineAuth | FIXED | `ip4` placeholder, `|| true` guards. |
| O5 / O6 / O7 | FIXED | `restart_dns` propagates failure into the restore branch; `realpath -m` compares the whole `conf.d` path; `readlink -f` matches relative links. |

## Blocking

### B1 · MAJOR · false "clean" — unreadable subdirectory inside a repo hides untracked data
`bin/fleet-audit:148` (`count_status`), and `bin/fleet-audit:354,397` (remote stderr only shown when NOT AUDITED).

**What goes wrong.** `git status --porcelain -z` on a repo containing a directory it cannot open prints
`warning: could not open directory 'secret/': Permission denied` on **stderr and exits 0**. `count_status`
only looks at the exit code, so the untracked files inside are invisible, `rc=0`, no ERR. The warning lands
in `$tmp/<id>.err`, which the laptop prints only when the host is NOT AUDITED. The row shows `ok` (or just
`UNLISTED`), the host is `audited`, exit 0. Under the README rule that copy is deletable.

**Concrete trigger (reproduced).** `mkdir secret && echo data > secret/f && chmod 000 secret` in a repo whose
HEAD is on `origin/main` → row `s14-unreadable … -` (clean). Real-world: a root-owned `0700` directory created
by a Docker bind mount, a `sudo`-run pipeline writing outputs, or a Slurm job as another user, sitting in
a repo and *not* gitignored. Lab workstations are exactly where this happens.

**Why Blocking.** It is a false clean under the README rule (data loss class), and it is the same bug class
A3 claims to have closed, just one level down.

**Fix.** Capture status stderr and treat any output as untrustworthy:
```bash
g "$d" status --porcelain -z … >"$tf" 2>"$tf.err" || rc=1
[ -s "$tf.err" ] && rc=1          # git warns (unreadable dir, etc.) but exits 0
rm -f "$tf" "$tf.err"
```
Independently, on the laptop print `$tmp/<id>.err` when it is non-empty even for an `audited` host (one line,
`!! <id>: scanner stderr: …`), so no remote warning is ever swallowed again.

### B2 · MAJOR (borderline) · hosts.yaml still drops hosts silently with exit 0
`bin/fleet-audit:79-80`:
```awk
line ~ /^[^ ]/ { flush(); inh=0; next }
!inh { next }
```
**What goes wrong.** Any line that does not start with a space (a host id dedented to column 0, a
tab-indented stanza, a stray top-level key) ends the `hosts:` section, and from then on **every line is
dropped by `!inh { next }` without calling `err()`**. The parser exits 0, the summary shows only the hosts
before the typo as `audited`, exit 0. The claim "an inventory line the parser doesn't understand is a config
error (exit 2), never a host that quietly drops out" is not met.

**Concrete triggers (all reproduced, all exit 0 with only `laptop` parsed).**
```yaml
hosts:
  laptop: …status: active
lab:                     # dedented by one editor mishap
    ssh: lab
    roots: ~/Projects
    status: active
  lab3: …                # also lost
```
Same with `\tlab:` (tab indent) and with `notes: foo` inserted between two stanzas (everything after it lost).
`projects.yaml` hosts referencing `lab` do print "lists unknown host id" on stderr, which is the only hint.

**Why Blocking (borderline).** A silently unaudited host is explicitly the worst class in the brief, and the
dedent happens exactly when the owner uncomments the `# lab2:` template for the three new machines. I put it
here rather than Non-blocking because the fix is two lines and the failure is silent on stdout and in the exit
code.

**Fix.** Once `hosts:` has been seen, treat any further top-level line as an error, and reject tabs:
```awk
line ~ /\t/ { err("line " NR ": tab indentation"); next }
line ~ /^[^ ]/ { flush(); if (inh) err("line " NR ": unexpected top-level line after hosts: " line); inh=0; next }
```
(If other top-level sections are ever wanted after `hosts:`, whitelist them explicitly.)

## Non-blocking

### N1 · MINOR · spurious ERR from `count_hidden` exit-status leak
`bin/fleet-audit:215`: `S\ *) [ -e "$d/${line#S }" ] && hid=$((hid+1)) ;;`. When the **last** `ls-files -v`
line is a skip-worktree entry absent on disk, the `&&` list returns 1, that is the loop's status, the
function returns 1, and the repo gets `ERR`. Reproduced: sparse checkout with `s` skip-worktree and removed
→ `ERR`; add a file sorting after it → clean. Safe direction (false positive), but ERR is a stop flag the
README says means "counts can't be trusted", so every sparse-checkout repo would stay undeletable and the
owner would learn to ignore ERR. Fix: end the function with `return 0`, or write the branch as
`if [ -e … ]; then hid=$((hid+1)); fi`.

### N2 · MINOR · `FLEET_AUDIT_FORGES` / `FLEET_AUDIT_GIT_SSH` are read on the *remote*, never forwarded
`bin/fleet-audit:131,138`. The scanner reads these env vars, but `ssh … bash -s` does not carry the laptop's
environment, so setting them changes the `ssh: local` host only. Also, with
`FLEET_AUDIT_FORGES="github.com "` (trailing space) `FORGES` contains a double space and
`*" $(url_host "$u") "*` matches an **empty** host, so a local-path remote counts as a backup (reproduced:
`origin=/tmp/…/s2.git` → clean). Not blocking: it needs a hand-set, malformed env var, and only on one host.
Fix: in `backup_remotes` require `[ -n "$h" ]` before the membership test, and either forward the two vars
on the ssh command line (`bash -s -- … ` with `FLEET_AUDIT_FORGES=… exported inside SCAN`) or drop them
from the help text.

### N3 · MINOR · `git status` warnings and other remote stderr are invisible on success
Same root cause as B1's second half, listed separately because it also hides things like
`warning: unable to access '.gitignore'`, submodule-foreach errors (`2>/dev/null` at line 265), and
`find` permission errors. Suggested fix is in B1.

### N4 · MINOR · stale documentation in the script header and README
- `bin/fleet-audit:7` and `:40` still say `--fetch` runs `git fetch --all --prune`; it now fetches each
  forge remote by name.
- `fleet/README.md:133-135` still defines UNPUSHED as "no remote-tracking ref contains" and recommends
  `git log --branches --tags HEAD --not --remotes`, which uses *all* remotes and therefore under-reports
  relative to the tool (a commit only on `refs/remotes/lab/*` is UNPUSHED to the tool but not to that
  command). Should be `git log --exclude='refs/remotes/*' --exclude=refs/stash --all --not --remotes=<forge>`.
- `fleet/README.md:138` "Lines the parser doesn't understand are reported on stderr, not silently dropped"
  is false for hosts.yaml today (B2) and will be "exit 2" once fixed.
Doesn't block: docs only, but the README is the deletion rule, so fix with B2.

### N5 · MINOR · `pushInsteadOf` is not checked
`bin/fleet-audit:175-185` compares the configured URL and `remote get-url` (fetch URL). With
`url.ssh://lab/.pushInsteadOf=https://github.com/`, pushes go to the lab box, and `git push` updates
`refs/remotes/origin/main` locally, so without `--fetch` the commit looks pushed to GitHub. `--fetch` (which
the README requires) resets the tracking ref from GitHub and exposes it, so this is not a false clean under
the rule. Cheap hardening: also require `url_host "$(g "$d" remote get-url --push "$r")"` to be a forge.

### N6 · MINOR · `fleet-onboard --authkey` loses the typed key on the reboot path
`bin/fleet-onboard:83-88,106-110`. The key is read before `pacman -Syu`; if the tun check then exits 1, the
EXIT trap deletes the key file and the owner must retype it after reboot. Not a safety issue. Fix: move the
tun check before the key prompt (check `/sys/module/tun || modprobe tun` first, upgrade, re-check), or print
"you will be asked for the key again".

### N7 · NIT · tun presence test assumes a modular TUN
`bin/fleet-onboard:106`. `/sys/module/tun` exists only for loaded *modules*; a kernel with `CONFIG_TUN=y`
and no module parameters has no such directory, so the script would say "reboot, then rerun" forever.
linux-cachyos has `CONFIG_TUN=m` (verified), so it does not bite today. `[ -e /dev/net/tun ]` is true in
both cases and is what tailscaled actually needs.

### N8 · NIT · cosmetics in the table
- URL column shows the raw URL for non-GitHub forges (`ssh://git@GitHub.com:443/me/s22.git`,
  `https://git@git.overleaf.com/abcdef0123`), which overflows the 24-char column. `bin/fleet-audit:239`.
- `SUBMODULE-NOBACKUP:<path>` with a space in the path splits into two flags at `flags=($fl)`
  (`bin/fleet-audit:364`); a glob in the path does not expand because the whole word is prefixed.
- `IGNORED:n ignored-size:0` for zero-byte ignored files (du says `0`, not `0K`).

## Checked and found correct

- `url_host`: every spoofing direction I tried (`http://github.com@lab/x`, `ssh://lab/github.com/x`,
  `lab:github.com:x`, `https://github.com.lab/`, `file://github.com/x`, `ssh://git@GitHub.com:443/…`)
  resolves to the real host or to "not a forge". Case folding works.
- `--exclude='refs/remotes/*'` uses slash-crossing wildmatch, so refs of gone/non-forge remotes under any
  depth are excluded; `--remotes=origin` matches `origin/feature/x`; `origin-old` does not leak into
  `--remotes=origin`.
- Remote names containing glob characters cannot exist via `git remote add`; hand-edited
  `[remote "a*b"]` makes `git remote` itself fail → `NOREMOTE ERR` (safe).
- Forge remote never fetched (no tracking refs) → every commit UNPUSHED (safe, correct).
- Fork with `origin` + `upstream` on GitHub: a commit only on `upstream/*` counts as saved, which is the
  intended meaning of "on a forge". Both remotes get fetched by name.
- Submodule with lab-only remote → `SUBMODULE-NOBACKUP:sub`; submodule with forge remote is fetched and
  counted separately; uninitialized submodules have nothing to lose.
- Empty repo (no commits) with a forge remote → clean, correct.
- `IFS=$'\x1f' read`: non-whitespace IFS does not collapse adjacent delimiters; spaces in `roots`,
  names and flags survive; `-r` keeps backslashes.
- `#ROOTERR` + `#DONE` + rc 0 → `INCOMPLETE`, exit 1 (tested). `ROOT-MISSING` stays info, which is fine
  because the deletion rule also requires the copy's own row to be present.
- `set -e`/pipefail in `fleet-onboard`: `under_ssh` failures return 1 cleanly; `[ "${pid:-0}" -gt 1 ]`
  terminates on empty/odd values; `dig … | grep -q` did not SIGPIPE in 100 runs (dig's output is a single
  buffered write); `tailscale status … || true` and `tailscale ip … || true` are inside `$( )`, so
  pipefail cannot abort the script.
- `/proc/<pid>/stat` parsing: `sed 's/.*) //'` is greedy, so a comm containing `) ` cannot shift the ppid
  field.
- `realpath -m` on a missing `conf.d` path works; `$home` symlinks resolve equally on both sides.
- The DNS switch is still guarded both before (getent + dig through the current resolv.conf + stub
  answers) and after (dig through the new resolv.conf), with restore of the exact original file
  (`cp -a` preserves a symlink). Nothing in the diff touches sshd configuration or stops sshd; `--force`
  semantics unchanged.
- Idempotency of the new onboard code: `modprobe tun || true`, `systemctl enable` twice, re-running after
  the "reboot" exit resumes at `NeedsLogin`.
- `bash` `[a-z]` in `case` is ASCII-only (`globasciiranges` on by default in bash 5), so `H` is not
  miscounted as assume-unchanged.

## Bottom line

**(a) Is `fleet-onboard` safe to run on the 3 machines?** Yes. I found no path that cuts the network,
locks the owner out, or breaks idempotency. The two notes (N6 key re-prompt, N7 built-in-TUN theoretical)
are usability, not safety.

**(b) Can `fleet-audit` output be trusted for deletion decisions under the README rule?** Not yet.
Two silent false-clean paths remain: B1 (an unreadable subdirectory inside a repo hides its untracked
files and still yields `ok`) and B2 (a dedented/tab-indented/stray line in hosts.yaml drops all following
hosts with exit 0). Both are small fixes (capture `git status` stderr; make post-`hosts:` top-level lines
an error). Everything else about the backup-evidence rewrite (forge allow-list, `--all` minus backup
tracking refs, per-remote fetch, HIDDENIDX, LFS, REGEN) held up under attack, and its failure modes are in
the safe direction.

**(c) Ready to merge into main?** Not as is. Fix B1 and B2, fold in N1 (spurious ERR will otherwise train
the owner to ignore a stop flag) and N4 (the README is the rule), then merge. N2, N3, N5 to N8 can follow.
