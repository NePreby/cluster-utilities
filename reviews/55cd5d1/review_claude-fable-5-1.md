# Review of `fleet` commit 55cd5d1 (round 8, final pass)

**Reviewer:** Claude Fable 5.1 (model id `claude-fable-5-1`), run 2026-10-24.
**Scope honoured:** only the README LFS / LFS-EXTERNAL bullets (`fleet/README.md:32-40`), the archive + copy
bash block (`fleet/README.md:66-90`), and the `count_lfs` object count (`bin/fleet-audit:293-320`).
Nothing else under `reviews/` was read.

**Method.** I did not work from the diff. I extracted `count_lfs` plus its helpers (`g`, `absent`, `tmpf`)
from `bin/fleet-audit` and ran it against 20 synthetic store layouts under bash 5.3 / GNU findutils 4.11 /
git-lfs 3.8. I ran the README copy block verbatim (placeholders substituted) against a fake transport
that executes the quoted remote string under a real `fish -c` (fish 4.9.3) from a separate remote HOME,
with real `rsync` 3.5.1 going through the same transport, and GNU tar 1.35 for the archive step.

## Verdict table

| Item | Verdict | Inputs tried (concrete) |
|---|---|---|
| G1 "archive the external store" option | **FIXED** | README text only. The option is gone; `LFS-EXTERNAL` → keep the copy, with the correct reason (tar stores links as links, `tar -d`/sha256 still pass). Cross-checked that every layout that would have produced a symlink-only archive is flagged `LFS-EXTERNAL` by the scanner (cases 2, 3, 4, 5, 11, 12, 13, 16 below). |
| G2 `COPY OK` with empty hash | **FIXED** | `$d` with no `SHA256SUMS` → refused before any remote command, remote dir count unchanged. `SHA256SUMS` containing `abc  myrepo.tar.gz` → refused, no remote mkdir. Regex rejects "", 63 chars, 66 chars (`hash\nx`), uppercase hex; accepts 64 lowercase hex. Local tar.gz appended to after `SHA256SUMS` → copied but refused on hash mismatch. |
| G3 non-atomic remote existence test | **FIXED** | Fresh name → `COPY OK`. Same name again → `mkdir: File exists`, refused, prior remote archive byte-identical afterwards. Remote dir reserved + 10-byte partial `tar.gz` (interrupted rsync) → refused; the README recovery (remove, re-run) then gives `COPY OK`. Remote `archives` is a regular file → `mkdir -p` fails, refused. |
| Scanner `find -H` count | **FIXED** | 20 layouts, listed in the false-clean section. Deep files at depth 4 counted; links at any depth, dangling or looping, set `LFS-EXTERNAL`; unreadable `objects/` or shard, `objects` dangling or self-looping → ERR. |

## Blocking

None.

I could not produce data loss or a false `VERIFIED`/`COPY OK` from the block as written, nor a false
clean from the new count. The reason the copy step resists shell-quoting attacks is structural: the only
thing that prints `COPY OK` is the remote's `sha256sum` output being a single line whose first field equals
this machine's 64-hex hash. No misquoting on a fish remote can manufacture that string without a file of
identical bytes existing there. Everything misquoting can do, it does in the safe direction (refuse).

## Non-blocking (most severe first)

1. **Unquoted `$b` in both remote command strings; the reservation can land on the wrong name.**
   `fleet/README.md:81` and `:83`. Trigger: a repo dir named `my repo` (or `rep(1)`). Observed under fish:
   `mkdir archives/lkj008-my repo-2026-10-24-XXXXXX` creates `archives/lkj008-my` and a stray
   `repo-2026-10-24-XXXXXX` in the remote HOME; rsync (3.2.4+ escapes remote args) then creates the
   real `archives/lkj008-my repo-…/` and copies correctly; `sha256sum` gets three args and the step
   prints `!! copy NOT verified`. With parens fish errors on command substitution before `mkdir`. Both
   outcomes are safe (no false OK), but the `mkdir` guard protects the wrong path, so a second copy with
   the same `$b` would not be refused, and junk dirs are left on the remote. Not blocking because the
   hash compare still gates `COPY OK`, and a same-`$b` collision needs the same hostname + same
   `mktemp` suffix. Fix (tested under fish 4.9 with both names, including a correct refusal on re-run):
   ```bash
   && ssh <other-host> "mkdir -p archives && mkdir 'archives/$b'" \
   ...
   && [ "$(ssh <other-host> "sha256sum 'archives/$b/<repo>.tar.gz'" | cut -d' ' -f1)" = "$h" ] \
   ```
   (single quotes inside the double-quoted string; bash expands `$b`, fish sees a quoted literal).

2. **Recovery note only covers the "mkdir succeeded" case; the other failure is left to guesswork.**
   `fleet/README.md:87-89`. Trigger: the user re-runs the copy lines on a `$d` that already got `COPY OK`
   (lost terminal scrollback, double-checking). Output is `mkdir: … File exists` + `!! copy NOT verified:
   don't delete`. The note says what to do if the failure came *after* a successful mkdir; here mkdir
   failed, and the nearest instruction is "remove it". Removing is harmless as long as the local `$d`
   exists (it must, or the step refuses before mkdir), so no data loss when followed literally, which is
   why this is not blocking. Fix: add one sentence: "If it says `mkdir: … File exists`, the remote already
   has that name. To re-check it, run only the `sha256sum` compare line against `$h`; if it matches, that
   is your `COPY OK`."

3. **`LFS-EXTERNAL` prose is slightly over-conservative (not wrong).** `fleet/README.md:34-37`. "Archiving
   the store separately doesn't help either" is true for symlinked objects/shards, but for an absolute
   `lfs.storage` pointing at a plain directory a separate archive would work. Keeping the copy is the
   safer rule, and the owner chose it; I only note that the stated reason doesn't cover that case. No
   change needed.

4. **Pre-existing, not introduced here: stale default store after `lfs.storage` is moved.**
   `bin/fleet-audit:296-301`. Trigger (case 19): `.git/lfs/objects/ab/cd/old` exists, then
   `git config lfs.storage /elsewhere` with an empty `/elsewhere/objects`. Result: `ok`, no LFS flag,
   no LFS-EXTERNAL. The bytes at the old location are inside `.git`, so the archive recipe would still
   capture them, and git-lfs itself can no longer see them; so this is a weak "content". Not in the
   diff and not blocking. If wanted later: when `lfs.storage` is set and `$gd/lfs/objects` also exists
   and is non-empty, count it too (or flag ERR).

## Can the new count produce a false clean?

Not that I could construct. Layouts run (`rc`, `lfs`, `lfsext` are the function's outputs):

| # | Layout | rc | lfs | ext | Right? |
|---|---|---|---|---|---|
| 1 | sharded files at depth 2 and depth 4 | 0 | 2 | 0 | yes, depth no longer capped |
| 2 | shard dir is a symlink to an outside dir with files | 0 | 0 | 1 | yes |
| 3 | one real object + one dangling link | 0 | 1 | 1 | yes |
| 4 | only a looping link (`ab/loop -> ..`) | 0 | 0 | 1 | yes, and find didn't fail |
| 5 | `objects` is a symlink to an outside dir | 0 | 1 | 1 | yes (containment test) |
| 6 | `objects` dangling symlink | 1 | - | - | ERR, yes |
| 7 | only a FIFO under objects | 0 | 0 | 0 | clean; a FIFO holds no bytes |
| 8 | shard dir mode 000 | 1 | - | - | ERR, yes |
| 9 | empty `objects/` | 0 | 0 | 0 | clean, correct |
| 10 | `objects -> ../../lfsdata` (inside repo) | 0 | 1 | 0 | correct: tar gets link + target |
| 11 | as 10, but an object inside is a link to outside | 0 | 0 | 1 | yes |
| 12 | absolute `lfs.storage` outside | 0 | 1 | 1 | yes |
| 13 | `lfs.storage = ../../ext` escaping the repo | 0 | 1 | 1 | yes |
| 14 | `lfs.storage` only in `.lfsconfig` | 0 | 0 | 0 | correct: git-lfs 3.8 refuses that key ("unsafe .lfsconfig keys were ignored"), store really is `.git/lfs/objects` |
| 15 | `objects` is a symlink to a regular file | 0 | 1 | 0 | odd but not a false clean |
| 16 | link to an outside dir at depth 5 | 0 | 0 | 1 | yes |
| 17 | `objects/` mode 000 | 1 | - | - | ERR, yes |
| 18 | `objects/` mode 111 (searchable, unreadable) | 1 | - | - | ERR, yes |
| 19 | store moved via config, old objects left in `.git/lfs` | 0 | 0 | 0 | the one gap, pre-existing, see non-blocking 4 |
| 21 | `objects -> objects` (ELOOP) | 1 | - | - | ERR, yes |

The design argument holds: a regular file anywhere below `objects/` is counted (`-type f`, no depth cap);
anything reachable only through a link is by definition under a symlink that `-type l` finds and turns
into `LFS-EXTERNAL`; anything `find` can't list makes it exit non-zero → ERR. A bind mount at a shard
would be counted as regular files and is physically under the repo path, so a repo archive includes it.
The only way to hold content that is none of file / link / listable-dir is a device or FIFO, which
holds no bytes.

Two things I checked around the count that could have undermined it and didn't: a symlinked repo dir
in a root is counted LOOSE and never scanned (`bin/fleet-audit:448-451`), so no row can be an archive
of a single symlink; and the SSH login menu is guarded by `status is-interactive`
(`bin/fleet-onboard:126`), so non-interactive `ssh host "sha256sum …"` output is clean (noise would
only cause a safe refusal anyway).

## Bottom line

**Ready to merge into main.** No blocking findings. The three round-7 fixes hold against the inputs
above, and the scanner change removes the depth cap without opening a false-clean path.

**The copy step is a sound design and should be kept, not replaced.** Its correctness does not rest on
the `mkdir` reservation or on rsync's quoting. It rests on comparing the remote file's sha256 with a
hash computed on this machine, gated by a check that the local hash is real. That is the right trust
anchor and it is independent of what shell the remote runs. The `mkdir` reservation is a cheap
never-overwrite guard on top. The one thing I would change before or right after merge is non-blocking
item 1 (single-quote `'archives/$b'` in both remote strings, two characters each), and optionally the
one-sentence recovery addition in item 2. Neither warrants replacing the step with something simpler;
a simpler step would most likely drop the reservation, which is the part that makes an interrupted or
repeated run harmless.
