# Review of `fleet` commit c3eeb4d (round 5) — Claude Fable 5.1 (`claude-fable-5-1`)

Scope: the diff 7594a87..c3eeb4d in `bin/fleet-audit` and `fleet/README.md`, read in full, plus
`fleet/hosts.yaml` / `fleet/projects.yaml` for context. `bin/fleet-onboard` not reviewed (out of scope).

Method: I did not reason from the diff alone. I extracted the scanner, built a throwaway fleet
(`ssh: local`, a fake forge reached through `FLEET_AUDIT_GIT_SSH` + `FLEET_AUDIT_FORGES`, so
`ls-remote`/`fetch`/`push` really run against bare repos), and ran `fleet-audit --fetch --all` on
constructed repos. Environment: bash 5.3, git 2.56, coreutils 9.12 (`stat` prints `cannot statx`),
`find` = bfs 4.1 (supports `-H`/`-printf`; CachyOS findutils behaves the same for what's used).

## Round-4 items

| Item | Verdict | What I tried |
|---|---|---|
| D1 LFS store not traversable / dangling or symlinked `objects` | **FIXED** | `.git/lfs` mode 000 → `ERR`; `objects` → `/nonexistent` → `ERR`; `objects` symlink to a real dir with 2 files → `LFS:2`; `lfs.storage=/abs/store4` with one unreadable subdir → `ERR`; parent dir mode 000 → `ERR`. No case produced `ok`. |
| D2 README let `git lfs push --all` clear LFS | **FIXED** | README now says keep the copy or verified tar; `lfs push` explicitly rejected with the two correct reasons (`lfs.url`, unreferenced objects). |
| D3 ENOENT matched inside the path; roots must start with `/` or `~` | **FIXED** | Dir literally named `x: No such file or directory` that exists → not absent; the same name nonexistent → absent; path with newline → absent; `LANG=LANGUAGE=fr_FR` → still matched (LC_ALL=C wins). hosts.yaml roots `/a b`, `./x`, `Projects` → exit 2 before any scan. |
| N1 empty advertisement became `^missing` | **FIXED** | Forge repo re-created empty → row shows `UNPUSHED:1`, no ERR, no `^missing`. |
| N2 cat-file errors ignored | **FIXED** | Locally corrupted the loose object the server advertises → `FETCHFAIL ERR`, stderr surfaced under `!! warnings`. Also: server advertising tags that point at a **blob** and a **tree** → `ok` (rev-list accepts `^blob`/`^tree` silently; the `tree\|blob` filter is correct, not a trap). |
| N3 README manual UNPUSHED uses ls-remote | **FIXED** (one caveat, non-blocking) | Ran the command verbatim on the blob/tree-tag repo: lists nothing, exit 0. With a dead remote it lists *every* commit (no pipefail) — over-reports, never under-reports. |
| N4 `git replace -d <sha>` | **FIXED** | `git replace -l` prints the replaced sha; `git replace -d <that sha>` works. |
| N5 tar recipe | **FIXED** (two caveats below) | Ran verbatim: `VERIFIED`; modified a file afterwards → `tar -d` exit 1, no VERIFIED; ran `tar -d` from the wrong cwd → exit 1 (safe direction). A unix socket (`.git/fsmonitor--daemon.ipc`) → "socket ignored" warning but still exit 0 / VERIFIED, which is acceptable (sockets carry no data). |
| N6 scanner temp files in one dir, cleaned on exit incl. timeout kill; awk prefix | **FIXED** | Private `TMPDIR`: after per-host `timeout` kill → 0 leftovers; after direct `SIGTERM` to the scanner → rc 143, last line is a data row (no `#DONE`), 0 leftovers; after SIGPIPE (closed stdout, the orphaned-remote analogue) → rc 141, 0 leftovers. |
| N7 wording | FIXED | n/a |

## Blocking

None. I could not construct a false `ok`, a silently unchecked host, or a silently unchecked entry
from any of the changed code paths, and no changed README instruction loses data when followed literally.

## Non-blocking

**MAJOR-ish / MINOR — per-host timeout is not honoured when the hang is inside an inner `timeout`.**
`bin/fleet-audit:138` (`trap 'exit 143' TERM`) + `:223` / `:241` (`timeout 90 git fetch`, `timeout 60 git ls-remote`).
GNU `timeout` puts its child in its own process group, so when the *outer* per-host `timeout` fires it
signals only the scanner bash and the scanner's group; the inner `timeout`+git are in a different group
and keep running. Bash defers a trapped signal until the foreground child exits, so the scanner only
executes `exit 143` after the inner 60/90 s limit. Trigger: `FLEET_AUDIT_TIMEOUT=3 fleet-audit --fetch`
with a hanging ssh for the forge: host result is `NOT AUDITED (timed out after 3s)` but wall-clock
is 90 s. With ~100 repos that's at most one overrun per host (after the first deferred TERM the
scanner exits), so this is latency and a misleading message, not a wrong verdict. Why not blocking:
the host is still reported NOT AUDITED and nothing is ever marked clean. Fix: either `timeout
--foreground` for the inner calls (they then share the scanner's group and die with it), or use
`-k 5` on the outer `timeout` in the main script (`:442`, `:447`) so a wedged scanner is KILLed; the
EXIT trap then won't run, but `$SCANTMP` on a wedged host is the lesser problem.

**MINOR — README tar recipe: files created *after* `tar -czf` are invisible to `tar -d`.**
`fleet/README.md:59-65`. `tar -d` only compares archive members with disk; a file added to the repo
between `tar -czf` and `tar -d` (an editor autosave, a running job writing into `data/`) still prints
`VERIFIED`, and the next step is deleting the repo. Trigger: run the recipe while a job writes to the
repo; archive lacks the new output; VERIFIED; delete. Why not blocking: the README's data-moving
section already says "stop anything still writing", and the user is about to delete the dir, so the
window is small. Fix: add "0. stop anything still writing to the repo" to the recipe, or do the check
the other way around (`tar -czf` then `find <repo> -newer "$a" | grep . && echo CHANGED-SINCE`), or
re-run `fleet-audit` after archiving and before deleting.

**MINOR — "copy the archive somewhere that isn't this machine" has no verification step.**
`fleet/README.md:65`. The whole point of `tar -d` is to not trust an unverified copy, then the copy
that actually survives deletion is unverified. Fix: `sha256sum "$a"` locally and on the destination,
or `rsync -c` (the same technique the "Moving gitignored data" section already mandates).

**MINOR — README manual UNPUSHED command silently over-reports when `ls-remote` fails.**
`fleet/README.md:158-161`. No `pipefail`, so a failed `git ls-remote` (no network, expired token)
yields an empty negation list and `git log --all` prints every commit in the repo as "unpushed".
Safe direction, but it will send the owner chasing phantom unpushed work. Fix: prefix the pipeline
with `set -o pipefail;` or run `git ls-remote origin >/tmp/adv || exit` as a separate first step.

**MINOR — `lfs.storage` set to an empty value counts the *git* object store as LFS.**
`bin/fleet-audit:286-287`. `git config lfs.storage ""` → `st=""` → `st="$gd/"` → `find -H .git//objects
-type f` counts every loose object and pack → `LFS:n` on a repo with no LFS at all. Tested: `LFS:2` on a
plain one-commit repo. False *stop* flag, so not a safety issue; it is also an unlikely config. Fix:
`[ -n "$st" ] || st=lfs` after the `config --get`.

**MINOR — `mktemp -d` failure is not fatal.** `bin/fleet-audit:138`. If `$SCANTMP` is empty, `tmpf`
falls back to `$TMPDIR`/`/tmp` (leaks, no cleanup) or fails; `count_unpushed` then writes `.in`/`.out`/
`.err` relative to the scanner's cwd (`$HOME` on a remote host). I traced every consumer: each failure
path ends in `ERR`/`bad=1`, so no false clean, but the row set is noisy and three stray files appear in
`$HOME`. Fix: `SCANTMP=$(mktemp -d) || { echo '#ROOTERR mktemp failed'; exit 1; }`.

**NIT — `awk -v d="$d"` interprets backslash escapes in the path prefix** (`:173`, `:247`). A repo dir
containing `\n` or `\t` is printed mangled in the warning line only; the `ERR` flag is unaffected.
Use `awk '{print d ": " $0}' d="$d"`? No, same issue; pass it via `ENVIRON` or `printf '%s: ' "$d"` per
line. Cosmetic.

**NIT — root validation accepts `~foo`, `~/Proj*`, `~/it's`.** `bin/fleet-audit:79`. All three are
harmless for the rule: `~foo` and `~/Proj*` (remote, single-quoted) become a visible `ROOT-MISSING` row,
`~/it's` on a remote host breaks the fish quoting and the host is NOT AUDITED. Locally `$roots` is
unquoted (`:442`), so an absolute root containing `*` glob-expands into several roots, each scanned under
its expanded name. If you want to tighten: `rs[i] ~ /^(\/|~\/|~$)/ && rs[i] !~ /['"*?\[]/`.

**NIT — `absent()` + `stat -- -`.** coreutils `stat` treats `-` as stdin, so `absent -` returns 1
(ERR). Unreachable now that roots must start with `/` or `~`.

**NIT — wording.** README `:32` says "`tar -df` check"; the recipe uses `tar -dzf`. `bin/fleet-audit:36`
help text lists ROOT-MISSING as info, which is right, but the README's "Before deleting" rule doesn't
mention it; it is covered implicitly by "the copy's row is present".

## Checked and found correct

- `absent()` (`:143-149`): ENOENT → 0; EACCES on a parent, ENOTDIR (regular file where a dir is
  expected), ELOOP (both the path itself and a parent), dangling symlink, FUSE-style errors → 1. The
  suffix match tolerates both `cannot stat` and `cannot statx`, the path containing the suffix text,
  quotes, newlines; `LC_ALL=C` overrides `LANG`/`LANGUAGE`. Exit status is ignored, which is right: a
  `stat` that unexpectedly succeeds gives an empty message → 1 → ERR, the safe direction.
- `count_lfs` (`:283-297`): the `absent → return 0`, `! -e → return 1`, `find -H` + `frc` sequence has no
  path to `0` except a definite ENOENT. An `lfs.storage` pointing at an unmounted volume reads as "no
  store"; that is correct because deleting the repo dir cannot lose content that lives on the volume.
- `count_unpushed` (`:236-261`): blank/odd advertisement lines dropped; `^{}` peeled lines contribute the
  peeled sha; `sha missing` lines filtered by the type regex; cat-file stderr or non-zero → `bad=1` →
  ERR even when `rc=2`; empty negation file → `rev-list --all` counts everything (N1); `^blob`/`^tree`
  accepted by rev-list and by `git log --stdin`. The `[ "$bad" = 1 ] && return 1` placement after
  `unp+=n` is correct (the count is still shown, flagged ERR).
- Traps: `TERM`→`exit 143`→EXIT trap verified to clean `$SCANTMP`; a TERM arriving during the final
  `echo '#DONE'` could print `#DONE` but rc is 143, and the "audited" test at `:476` requires both
  rc 0 **and** `#DONE`. Nested `$(...)` subshells reset the trap, so a TERM there kills the subshell and
  the parent's deferred trap still fires. Inner `timeout` processes don't get the outer TERM (see
  the MAJOR/MINOR item) but the final verdict is unaffected.
- hosts.yaml: split on `" "` in awk collapses runs of whitespace, so `roots:   ~/a   ~/b` is fine; a
  space in a path is rejected with a clear message and exit 2 before any scan; trailing comment is
  stripped first. Re-verified the unchanged parser still rejects tabs / column-0 ids.
- README: LFS instruction is now consistent with the tool (flag never clears; archive or keep); `tar -d`
  run from the parent dir compares correctly and exit 1 on any content difference; archive location
  `~/archives` is outside both roots in `hosts.yaml`; `git replace -l/-d` usage is correct;
  `--exclude` flags precede `--all` in both the tool and the README command, as required.
- Warning surfacing (`:479-482`): the corrupt-object stderr lines appear under `!! warnings` and the row
  carries `ERR`; `head -5` truncation is fine since the row has the flag.

## Bottom line

**(a) Is fleet-audit trustworthy for deletion under the README rule?** Yes, for everything in this
diff. Every error path I could construct in the changed code (unreadable or missing LFS stores, empty
or odd forge advertisements, corrupt objects, timeouts, signals, bad roots) ends in `ERR`, `FETCHFAIL`,
`ROOT-MISSING`, `NOT AUDITED`, or a conservative over-count. I found no false `ok` and no host or entry
that drops out silently.

**(b) Ready to merge into main?** Yes. No Blocking findings. Priorities after merge, in order:

1. Make the per-host timeout actually bound wall-clock (`--foreground` on the inner `timeout`s, or
   `-k` on the outer) so a wedged forge doesn't stall a run by 90 s per host and the "timed out after
   Ns" message tells the truth.
2. README tar recipe: add "stop anything writing" and verify the off-machine copy by checksum.
3. README manual command: `set -o pipefail` so a dead remote doesn't list the whole history as unpushed.
4. `lfs.storage` empty-string and `mktemp -d` failure hardening (both one-liners).
