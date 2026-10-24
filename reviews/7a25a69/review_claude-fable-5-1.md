# Round-2 code review: `fleet` branch @ 7a25a69 — Claude Fable 5.1

**Reviewer:** Claude Fable 5.1 (model id `claude-fable-5-1`), reviewing independently; I read only
`REVIEW_REQUEST.md` under `reviews/`.

**Method.** I read `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/hosts.yaml`, `fleet/projects.yaml`,
`fleet/README.md`, `bin/harden-sshd-keyonly`, `bin/cluster-status` in full. I then extracted the embedded
`SCAN` script and ran it (and the whole `fleet-audit`, from a copied tree with edited YAML, never the
originals) against a fixture tree of ~25 repos built to hit each claimed fix: detached HEAD, tag-only
commit, notes-only, reflog-only, submodule with unpushed work, linked worktree with untracked file,
remote section without `url`, `user@host:path` and `ssh://host/` fleet-style remotes, unborn HEAD,
single-branch refspec, ignored data with spaces and REGEN-named dirs, nested repos at depth 1 and 5,
a bare repo, an empty dir, a `chmod 000` dir, loose files, a symlink, a dot-dir, an `assume-unchanged`
file, an unreachable `--fetch` remote, a GitHub-style remote (for the `ok` row), an unresolvable ssh
host, a `FLEET_AUDIT_TIMEOUT=0.05` run, and the remote invocation handed to a real `fish` the way ssh
would hand it. Environment: CachyOS, bash 5.3.20, git 2.56, gawk 5.4, fish and tailscale 1.102
present. `shellcheck` is not installed here. `sudo` needs a password here, so the one
`fleet-onboard` finding about `sudo` stripping `SSH_CONNECTION` is from sudoers semantics, not a live
run; I say so where it matters.

## Claimed fixes

| # | Claim | Verdict | What I tried / saw |
|---|---|---|---|
| 1 | UNPUSHED over branches+tags+HEAD, DETACHED flag | **HOLDS** | Detached HEAD with one extra commit → `DETACHED UNPUSHED:1`. Tag on a commit the branch was reset away from → `UNPUSHED:1`. Notes-only and reflog-only commits are *not* counted (by design; see N6). |
| 2 | Submodules + worktrees audited and counted in | **HOLDS** | Submodule with a local commit → parent row `UNPUSHED:1 DIRTY:1 SUBMODULES:1`. Linked worktree with a new file → `UNTRACKED:1 WORKTREES:1`. Recursive `foreach` prints correct absolute paths. |
| 3 | Untracked listed regardless of `status.showUntrackedFiles` | **HOLDS** | `--untracked-files=normal` is on the command line; CLI overrides config. |
| 4 | IGNORED:n minus regenerables; `--size` adds du | **PARTIAL** | Count is right (`data/` + `run 1.log` → `IGNORED:2`; `.cache/`, `env/` skipped). `--size` reported `ignored-size:4.0K`, i.e. only `data/`: porcelain v1 quotes `"run 1.log"` and `du` is fed the quoted string (N4). REGEN silently drops real data under `.cache/`/`env/` (N5). |
| 5 | `--fetch` → FETCHFAIL; LOCALREMOTE for path / host:path using configured URL | **PARTIAL** | Unreachable remote → `FETCHFAIL`, exit still 0 for the host. `lab:Projects/x` and `/tmp/...` → `LOCALREMOTE`. But `yorguin@lab:Projects/scp` and `ssh://lkj008-psy/...` → **no flag** (B3). |
| 6 | Every git call checked → ERR; `--no-optional-locks` | **HOLDS** | Remote section with `fetch` but no `url` → `UNPUSHED:1 ERR` (and the UNPUSHED count survived). `$?`-in-`else` pattern verified to see the condition's status (1 vs 128). |
| 7 | Dot-dirs, LOOSE:n, EMPTY, ROOT-MISSING, nested repos | **PARTIAL** | `.dotproj` listed, `LOOSE:2` (file + symlink), `EMPTY`, `~/code` → `ROOT-MISSING`, `kirby` → `NOGIT NESTED:1` + `kirby/katam NOREMOTE`. But a `chmod 000` dir prints `EMPTY` (N2); a repo at depth 5 is missed, parent stays NOGIT (N7); a bare repo prints `NOGIT` (safe). |
| 8 | `#DONE` sentinel; NOT AUDITED / INCOMPLETE + exit 1; unknown/inactive → 2 | **PARTIAL** | Unresolvable host → `!! lab: NOT AUDITED (ssh failed): …`, exit 1, not-found check skipped. Timeout → `INCOMPLETE … after 2 entries`, exit 1. `fleet-audit nope` → 2; retired host → 2. **But** an *active* host whose `roots:` line is mistyped is silently skipped and the run exits 0 (B1). |
| 9 | YAML lines normalized; unparsed/duplicates reported; `[*]` | **PARTIAL** | `  laptop:   # me`, `status: Active \r` (CRLF), quoted name, `[*]`, duplicate (reported) all fine. A 4-space-indented project line is dropped **without** a report; `[Lab]` is accepted and never matches (N3). |
| 10 | resolv.conf switch guarded, verified, restored, rerun re-checks | **HOLDS** (caveat) | Logic traced: no backup → no change (`set -e`), restore path correct, rerun with stub+broken DNS exits 1 with undo hint. Caveat: `dns_ok` goes through nss-resolve, not `/etc/resolv.conf` (N8). On this CachyOS box `/etc/resolv.conf` already points at the stub, so step 4 is likely a no-op on the new boxes. |
| 11 | Auth key hidden → 0600 temp file → `--auth-key=file:` | **HOLDS** | `read -rs </dev/tty`, `mktemp`+`chmod 600` as root, `unset key`, EXIT trap. `tailscale up --help` here documents `--auth-key` and says it's not persisted, so passing it on an already-logged-in node is harmless. |
| 12 | Menu installed as the user; 0700 `~/.config` kept; symlink dest left alone | **HOLDS** | `mkdir -p`/`tee` via `sudo -u`. Idempotency compare (`$(cat)` vs `read -d ''` variable) verified equal. Only the leaf symlink is checked (N11). |
| 13 | `-Syu`; BackendState; refuses over SSH w/o `--force`; refuses root; sudo via bash+abs path; `--help` w/o sudo | **BROKEN** for the SSH refusal, rest HOLDS | The `SSH_CONNECTION` test at line 53 runs *after* `exec sudo` at line 34; `sudo`'s default `env_reset` drops `SSH_*` (B2). Root login refused via `SUDO_USER`; `readlink -f "$0"` correct; `--help` exits before sudo; `BackendState` sed matches the real multi-line JSON (`"BackendState": "Running",`). |
| README | Deletion rule matches flags; checksum staging recipe | **PARTIAL** | Flag list matches the header. Recipe (`rsync -aP` to staging, `rsync -anci` verify, manual merge) is correct. Missing: UNDECIDED/EMPTY/`ignored-size:` not placed in either list; REGEN exclusions and LOCALREMOTE detection limits not stated (N5, N14). |

## Blocking

### B1 · MAJOR · `bin/fleet-audit:60-84` — an active host with a malformed `roots:`/`ssh:` line is silently skipped and the run still exits 0
**Class:** false "every host audited" (a host silently not checked), the class the brief calls most serious.

**What goes wrong.** awk prints `id\tssh\troots\tstatus`; when a field is empty the line has consecutive tabs, and `IFS=$'\t' read` (line 77) treats tab as IFS whitespace, so **adjacent tabs collapse** and later fields shift left. For `lab2` with `ssh: lab2`, a mistyped `Roots: ~/Projects` (reported as unrecognized, fine) and `status: active`, awk emits `lab2\tlab2\t\tactive`, bash reads `roots=active st=""`, line 81 prints `(skipping host 'lab2': status <none>)` to **stderr**, and the "needs ssh: and roots:" exit-2 check on line 82 is never reached. Nothing about lab2 appears on stdout, `ran` doesn't include it, and `exit "$failed"` is 0.

**Concrete trigger (reproduced).** hosts.yaml stanza:
```
  lab2:
    ssh: lab2
    Roots: ~/Projects        # or 3-space indent, or `roots` missing
    status: active
```
→ stderr `hosts.yaml:12: unrecognized line, ignored: …` + `(skipping host 'lab2': status <none>)`; stdout ends `== laptop: audited (…)`, exit 0. The same collapse makes a `planned` host without `roots:` show as `status <none>`, and `fleet-audit lab3` says *unknown host id* instead of *not active*.

**Why this matters for the README rule.** Rule 3 says a run that "ends with every host `audited` and exits 0" is the precondition. This run satisfies it with a host unaudited; the only trace is a stderr line that scrolls off (or is lost under `2>/dev/null`/a pipe). This is exactly the moment the three new boxes are added by hand to hosts.yaml.

**Fix.** (1) Never emit empty fields: in the awk `flush()`, print `(ssh==""?"-":ssh)` etc., and treat `-` as empty in bash; or switch the delimiter to a non-whitespace byte (`\x1f`) and `IFS=$'\x1f'`. Same for `proj_tsv`. (2) Make an active-but-unusable host fatal (it already is once the fields line up). (3) Print skipped hosts in the stdout summary (`-- lab2: skipped (status planned)`), so the "every host" claim is visible where the rule is checked. (4) Treat an empty `status:` as a config error, not as "inactive".

### B2 · MAJOR · `bin/fleet-onboard:32-57` — the "refuse over SSH without `--force`" guard can never fire
**Class:** removes the protection against cutting the machine off / leaving it half-done. Borderline (the owner's documented procedure is at the keyboard), placed here because the claimed fix is non-functional and the cost of the fix is a few lines.

**What goes wrong.** Line 34 `exec sudo -- bash … "$@"` happens first. `sudo`'s default `env_reset` keeps only its built-in `env_keep` list (`DISPLAY`, `PATH`, `LANG`, `XAUTHORITY`, …); `SSH_CONNECTION`, `SSH_CLIENT`, `SSH_TTY` are not in it on Arch/CachyOS unless the owner added them. By line 53 the variable is gone and the check is a no-op. **Not verified live here** (sudo needs a password in this sandbox); verify on `lab` with `ssh lab 'sudo env | grep -c SSH_'` (expect 0).

**Concrete trigger.** `ssh lab2 'bash Projects/cluster-utilities/bin/fleet-onboard lkj009-psy'` (what someone does when the keyboard is in another room) → sudo prompts, script proceeds, restarts `NetworkManager` and `tailscaled` mid-session. An NM restart on a wired desktop normally survives, a `tailscaled` restart drops a Tailscale-routed ssh for a few seconds, so a cut is *possible* rather than certain, but the point of the guard was to not depend on that.

**Fix.** Evaluate `--force` and `SSH_CONNECTION` *before* the `exec sudo` (parse the flags once at the top; the pre-sudo copy of the script sees the user's environment), or re-exec with `sudo --preserve-env=SSH_CONNECTION,SSH_TTY`. Belt and braces: also test `[ -n "${SSH_TTY:-}" ]` or `loginctl show-session "$(loginctl session-status 2>/dev/null | head -1 | awk '{print $1}')" -p Remote`.

### B3 · MAJOR (borderline) · `bin/fleet-audit:147-151` — LOCALREMOTE misses `user@host:path` and `ssh://host/path` fleet remotes
**Class:** false clean under the README rule, which says LOCALREMOTE "means it holds something not saved elsewhere". Borderline because the data in that case sits on another fleet host that is itself audited and would show the commits as `UNPUSHED` there; data is only lost if both copies are deleted in one pass. Placed here per the brief's instruction for borderline false-clean paths.

**Concrete trigger (reproduced).** Repo with `origin = yorguin@lab:Projects/scp` and `other = ssh://lkj008-psy/home/y/Projects/scp` → row `UNPUSHED:1 UNLISTED`, **no LOCALREMOTE**. The `*@*:*` arm (line 149) is meant for `git@github.com:` but matches any scp-style URL with a user, and the `*://*` arm (line 148) only special-cases `file://`. Both forms are what you get from `git remote add lab yorguin@lab:Projects/x` or `git clone lab:Projects/x` when `~/.ssh/config` has `User`.

**Fix.** Pass the fleet identifiers into `SCAN` (ssh aliases, `tailscale:` names, `ip:` values from hosts.yaml, plus `localhost`/`127.*`/`100.64-127.*`) and flag a remote when its host part is one of them, after stripping `user@`/scheme. Alternatively invert the logic: only `https://`/`ssh://`/`git@` URLs whose host is a known forge (`github.com`, `gitlab.com`, …) are "real"; everything else is LOCALREMOTE. The owner's remotes are GitHub, so the allow-list is simplest and safest.

## Non-blocking

### N1 · MAJOR · `bin/fleet-audit:262-263` — the row parser has the same tab-collapse latent bug; an empty field turns any flags into `ok`
Reproduced in isolation: `printf 'r\tn\tmain\t\t-\tERR UNPUSHED:3\n' | IFS=$'\t' read -r root n br url sz fl` gives `url=-`, `sz="ERR UNPUSHED:3"`, `fl=""` → `flags=()` → row prints `ok` (flags visible only in the SIZE column), or is hidden entirely without `--all`. I could **not** make `emit` produce an empty field with git 2.56 (`get-url` on a url-less remote prints the remote name; `du -sh` on an unreadable dir still prints `0`; `symbolic-ref --short` is never empty), which is why this is non-blocking. It is one git behaviour change away from a false clean. **Fix:** in `emit`, substitute `-` for empty args (`"${1:--}"` …), or use the `\x1f` delimiter as in B1.

### N2 · MAJOR · `bin/fleet-audit:209-212` — an unreadable or non-traversable dir is reported `EMPTY`
Reproduced with `chmod 000 ~/Projects/noperm` → `EMPTY`. `rev-parse` fails, `ls -A` prints nothing (stderr hidden), `-z` is true. **Why not blocking:** EMPTY is not in the README's allowed-flag list, so the rule as written still forbids deletion, and dirs under the owner's own home are normally readable. **Fix:** before the `ls -A` test, `[ -r "$d" ] && [ -x "$d" ] || { emit "$root" "$n" - - "$sz" "ERR NOACCESS$badname"; return; }`.

### N3 · MINOR · `bin/fleet-audit:91-98` — projects.yaml: over-indented entries are dropped silently; host ids are not validated
`    tagged: [laptop]` (4 spaces) matches neither the entry regex (line 93) nor the "unrecognized" regex `^  [^ #]` (line 98) because its third character is a space; nothing is printed. `notes: [Lab]` is accepted and yields `MISPLACED(owner:Lab)` on every host and never "registered … but not found". **Why not blocking:** both produce extra warnings (UNLISTED/MISPLACED), never fewer. **Fix:** report `^ {3,}[^ #]` lines; after parsing, warn on any host id in `proj_tsv` that is not `*`, `?` or a key of `known`.

### N4 · MINOR · `bin/fleet-audit:115-121,201` — `ignored-size` ignores paths that porcelain v1 quotes
With `core.quotePath` default, `!! "run 1.log"` arrives quoted; `ignpaths` gets `…/"run 1.log"`, `du` fails on it silently. Reproduced: expected size of `data/` + `run 1.log`, got `4.0K` (data only). Count stays correct. **Fix:** `g "$d" -c core.quotePath=false status …` (still not safe for newlines; use `-z` + `read -d ''` if you want it exact), or parse the quoted form.

### N5 · MINOR · `bin/fleet-audit:108` + `fleet/README.md:25-26` — REGEN excludes `.cache/`, `env/`, `venv/` contents from IGNORED without telling the reader
Reproduced: `.cache/realdata.h5` and `env/secret` → no IGNORED. `.cache` is where HF/torch put caches but also where people stash intermediate results; `env` is a plausible data or config dir name. **Why not blocking:** documented in the script header and an explicit design choice. **Fix:** list the REGEN names in README rule 3 ("these are assumed regenerable and not counted"), and consider dropping bare `env` (keep `.env`-less venv names `.venv`/`venv`).

### N6 · MINOR · `bin/fleet-audit:128-133,154` — remaining UNPUSHED blind spots, with a judgement on each
Reproduced as "ok" in fixtures: (a) **notes-only** (`git notes add`) — refs/notes is outside `--branches --tags HEAD`; rare for this owner, skip. (b) **reflog-only commits** (amend/reset) — by definition discarded work; recovering them is a feature, not a backup; skip. (c) **`assume-unchanged`/`skip-worktree` edits** — `status` hides them; rare, skip. Not reproducible but real: (d) **stale `refs/remotes/<name>/…` of a remote that no longer exists in config** (hand-edited `.git/config`) — counted as pushed, and `fetch --prune` only prunes configured remotes; rare. (e) **replace refs / `refs/original/`** after `filter-branch`; rare. Sparse checkout, partial clone and LFS: nothing local is lost (LFS objects are pushed by the pre-push hook; if that failed the push failed). **Cheap improvement** covering (a),(d),(e): `git rev-list --count --exclude='refs/remotes/*' --exclude='refs/stash' --all --not $(git for-each-ref --format='%(refname)' $(git remote | sed 's|^|refs/remotes/|'))` — i.e. "everything except stash, minus the refs of *configured* remotes". The single-branch-refspec case I expected to be a false clean was not: `push` doesn't create a remote-tracking ref outside the fetch refspec, so the branch shows `UNPUSHED` (a safe false positive).

### N7 · MINOR · `bin/fleet-audit:214-215` — nested-repo find: depth > 4 missed, repo-in-nested-repo double counted
`find -mindepth 2 -maxdepth 4 -name .git -print0 -prune` prunes the `.git` dir only, not the repo containing it, so `kirby/katam/vendor/x/.git` would be a second NESTED entry (and `katam`'s own status would already show `vendor/` as untracked). A repo at depth 5 (`kirby/deep/er/deeper/repo5`) is not listed. **Why not blocking:** the parent dir is still `NOGIT NESTED:n`, which forbids deletion. **Fix:** `find "$d" -mindepth 2 -maxdepth 6 \( -name .git -prune -print0 \) -o \( -type d -exec test -e '{}/.git' \; -prune \)` is awkward; simpler: after collecting, drop any nested path that has another nested path as a prefix.

### N8 · MINOR · `bin/fleet-onboard:59,156-168` — the DNS probes test nss-resolve, not `/etc/resolv.conf`
On this CachyOS box `/etc/nsswitch.conf` has `hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns`, so `getent hosts` asks `systemd-resolved` over varlink and **never reads `/etc/resolv.conf`**. "DNS works before" and "DNS still works" are therefore true regardless of what the symlink points at. The only consumers that would break are programs reading resolv.conf directly, which includes every Go binary (tailscale itself) and anything in containers. With the default `DNSStubListener=yes` (as here) the stub answers and nothing breaks, hence non-blocking. **Fix:** also require the stub listener to be up before switching: `ss -lun | grep -q '127.0.0.53'` (note the socket shows as `127.0.0.53%lo:53`), or `resolvectl query -p dns archlinux.org`, or a direct probe: `getent` can't do it, but `python -c` / `drill @127.0.0.53` can if present.

### N9 · MINOR · `bin/fleet-onboard:81-83,182-199` — `pacman -Syu` may upgrade the kernel; `tun` is a module on CachyOS, so tailscaled can fail until reboot
Verified here: `CONFIG_TUN=m` on `linux-cachyos`. On a fresh install that has never run tailscaled, `tun` is not loaded; `-Syu` installs a new kernel and removes `/usr/lib/modules/$(uname -r)`; `systemctl enable --now tailscaled` "succeeds" but tailscaled can't create `tailscale0`, `tailscale status --json` stays empty for 20 s, `tailscale up` errors, `set -e` exits 1 before the hand-off text. Rerunning after the reboot works, so this is a usability trap, not a safety one. **Fix:** `modprobe tun 2>/dev/null || true` *before* `pacman -Syu`; after it, `[ -d /usr/lib/modules/$(uname -r) ] || { echo "kernel upgraded: reboot, then rerun fleet-onboard"; exit 1; }`.

### N10 · MINOR · `bin/fleet-onboard:191,202` — `NeedsMachineAuth` path dies before the hand-off
`tailscale ip -4` has no address yet → non-zero → `set -e` → no "paste this on the laptop" block, and the auth-key temp file is already removed (fine). **Fix:** `ip4="$(tailscale ip -4 2>/dev/null | head -1 || true)"; ip4=${ip4:-<pending approval>}`.

### N11 · MINOR · `bin/fleet-onboard:123-133` — only the leaf is checked for a symlink
stow/chezmoi usually symlink `~/.config/fish` or `~/.config/fish/conf.d`; `mkdir -p` and `tee` follow the link and write the file into the dotfiles source tree (which then shows as UNTRACKED in that repo's audit, so it is at least noticed). **Fix:** `[ "$(readlink -f "$(dirname "$dest")")" = "$home/.config/fish/conf.d" ] || skip`.

### N12 · NIT · `bin/fleet-onboard:145` — relative symlink not recognized
systemd's own docs install `/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf`; `readlink` returns the relative form, the branch falls to "switch", and every run makes a new backup and relinks (harmless, same target). Use `readlink -f`.

### N13 · NIT · `bin/fleet-audit`
Line 140: `p` is not declared `local` in `scan_repo` (used at 171/179); harmless since the scanner is a child process. Line 275: `sort -k1,1 -k2,2` splits on whitespace, so a dir name with spaces sorts by its first word. Line 229: a *symlink to a directory* in a root counts as LOOSE and is never audited; intentional per the header, but a symlinked repo dir (`~/code/foo -> ~/Projects/foo`) would hide behind `LOOSE:n` — fine for safety, worth a word in the header.

### N14 · MINOR · `fleet/README.md:19-27, 106-111`
`UNDECIDED`, `EMPTY` and `ignored-size:` appear in neither the allowed nor the forbidden list; state "UNDECIDED = decide the owner first, don't delete" and "EMPTY = `rmdir` is fine". Mention the REGEN names (N5) and that LOCALREMOTE currently only recognizes bare `host:path` and filesystem paths (B3). The recipe is correct; add that the `_incoming-from-laptop/` dir will show as `UNTRACKED:1` on the owner host until merged, which is the intended reminder. Rule 3's first bullet should also say "and no `(skipping host …)` line on stderr" until B1 is fixed.

## Checked and found correct
- `#DONE` sentinel, `rc` file, INCOMPLETE vs NOT AUDITED wording, exit 1, and skipping the not-found check for unaudited hosts (tested both ssh-failure and timeout paths).
- Unknown host → exit 2; inactive host → exit 2; duplicate host id → exit 2; duplicate project → reported, first wins.
- Inline comments on host-id lines, CRLF line endings, `Status: Active`, quoted project names, `[*]`, `[?]`, `[]` all parse as intended.
- The `if br=$(…); then :; else [ $? -eq 1 ]` pattern: `$?` inside `else` is the condition's status (verified 1 vs 128).
- Background subshells get `/dev/null` stdin, so the local `bash -c "$SCAN"` cannot eat the hosts list; the ssh path feeds the script via `<<<`; `fetch` has `</dev/null`.
- Remote invocation through fish with single-quoted roots: fish passes `~/Projects` literally, the scanner expands it with the remote `$HOME` and reports it as written (tested with a real fish).
- `--fetch`: BatchMode + `GIT_TERMINAL_PROMPT=0`, 90 s timeout, failure → `FETCHFAIL`; `--prune` makes deleted-on-GitHub branches surface as `UNPUSHED`.
- `rev-list --count` with no refs at all (unborn HEAD) → 0, no error; `get-url` on a url-less remote → `ERR`.
- `count_status` with `--ignore-submodules=none` counts a modified submodule as DIRTY; `--ignored=matching` shows `data/` once.
- `fleet-onboard`: `--help` before sudo; `readlink -f "$0"` makes `bash fleet-onboard` from any cwd work; root login refused; auth key never in argv; `set -e` ensures no backup → no resolv.conf change; restore path restores the symlink or file with `cp -a`; `tailscaled` restarted only when resolv.conf changed; menu compare is byte-stable across reruns; nothing touches sshd auth.
- README recipe: `rsync -aP` into a staging dir cannot overwrite; `rsync -anci` is a true checksum dry-run that prints nothing on identity.

## Bottom line

**(a) Is `fleet-onboard` safe to run on the 3 machines?** Yes, **at the keyboard, as written.** Nothing in it can lock the owner out (sshd auth untouched, password login stays on) or leave the box without DNS (the backup/verify/restore path is correct, and on CachyOS the stub symlink is likely already in place so step 4 is a no-op). The worst outcomes are an exit 1 before the hand-off text (N9 kernel/tun after `-Syu`, N10 pending machine approval), both fixed by a reboot and a rerun. Do **not** rely on the over-SSH refusal (B2): it does not work; if you ever run it remotely, know that you are implicitly `--force`. Fix B2 (move the check above `exec sudo`) and add `modprobe tun` before the upgrade; both are five-line changes, and I'd do them before the first box.

**(b) Can `fleet-audit` output be trusted for deletion decisions under the README rule?** **Not yet, for one reason: B1.** The rule's foundation is "every host audited, exit 0", and today a one-character typo in a new host stanza makes the tool satisfy that while skipping the host, with only a stderr line to say so. That is precisely the situation about to occur when three stanzas are typed in. Fix B1 (placeholders for empty fields, skipped hosts in the stdout summary), then the per-row flags are trustworthy with the documented caveats: UNPUSHED/DIRTY/UNTRACKED/STASH/IGNORED/SUBMODULES/WORKTREES/DETACHED/FETCHFAIL/ERR all behaved correctly against adversarial fixtures. B3 (fleet remotes with `user@` or `ssh://` not flagged) should be fixed too, but given all of the owner's remotes are GitHub it can be covered in the meantime by eyeballing the URL column for anything that isn't `owner/repo`. N1 and N2 are the same shape of bug as B1 and should go in the same commit. Everything else is polish.

Priority order: **B1 → B2 → N1/N2 (same patch as B1) → B3 → N9 → N5/N14 (README wording) → the rest.**
