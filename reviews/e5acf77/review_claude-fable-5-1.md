# Review of `fleet` fixes at e5acf77 (round 7, scoped to 55ebe14..e5acf77)

**Reviewer:** Claude Fable 5.1 (model id `claude-fable-5-1`), Anthropic. Independent review; no other file
under `reviews/` was read.

**Method.** I read `bin/fleet-audit` in full (focus on `count_lfs`, lines 291-317, and the driver at
455-456) and `fleet/README.md` in full. I extracted the per-host scanner heredoc from `bin/fleet-audit`
and ran it unmodified against 15 synthetic LFS stores (symlinked shards, symlinked object files,
dangling links, loops, links to `/usr` and `/proc`, hard links, an unreadable shard, an absolute
`lfs.storage` with links inside). I ran the README's archive + copy chain literally with the remote
side emulated by `fish -c "<cmd>"` (what sshd does for a fish login shell), and the README's manual
UNPUSHED command against a failing and a working origin. GNU findutils 4.11, tar 1.35, git-lfs 3.8.

## Round-6 items

| Item | Verdict | What I tried |
|---|---|---|
| F1 symlinks below `objects` invisible | **FIXED** | all-linked shard → `LFS:1 LFS-EXTERNAL`; mixed real+linked object → `LFS:2 LFS-EXTERNAL`; only a dangling link → `LFS-EXTERNAL` (no `LFS:n`, not `ok`); dangling link + real objects → `LFS:1 LFS-EXTERNAL`; loop `objects/aa -> .` and `objects/aa -> ../../..` → `ERR`; link to an empty external dir → `LFS-EXTERNAL`; unreadable shard → `ERR`; dangling `objects` symlink → `ERR`; absolute `lfs.storage` whose store holds a linked shard → `LFS:1 LFS-EXTERNAL`; hard-linked object → `LFS:1` only (correct: tar stores content). No case produced a bare `LFS:n` or `ok` for a store with links. |
| F2 archive copy check could approve a different archive | **FIXED** (one narrow gap, see N1) | fresh fish remote → `COPY OK`; same name already on remote (also the "`<other-host>` is this machine" case) → refused, `NOT verified`; local tar.gz altered after SHA256SUMS → `NOT verified`; remote without `sha256sum` → `NOT verified`; local `SHA256SUMS` missing **and** remote file missing → false `COPY OK` (N1). |

## Blocking

### B1 · MAJOR (borderline, placed here per the brief) · `fleet/README.md:36-38` — "archive the store itself with the same recipe" produces a verified archive that holds only links

**What goes wrong.** The new text tells the user to resolve `LFS-EXTERNAL` by running the archive recipe
on the store directory. But the recipe is `tar -czf … <dir>` + `tar -dzf`, which stores symlinks as
symlinks. Two of the four causes the README itself lists for `LFS-EXTERNAL` are "a symlinked `objects`
dir" and "symlinked shards/objects inside the store", and for those the store archive contains zero
object bytes. `tar -d` passes (the link in the archive matches the link on disk), `sha256sum` passes,
`VERIFIED` and `COPY OK` print, and the README then says "The objects then have their own verified
copy". They do not. This is the exact mechanism round 6 flagged for the repo archive (the author's own
comment at `bin/fleet-audit:308-309`: "tar archives the link, not the content"), now re-introduced one
paragraph lower for the store archive.

**Concrete trigger (reproduced).** Store `lfs/objects/aa -> /tmp/disk/aa` with `/tmp/disk/aa/bb/obj`
present. `cd <parent>; tar -czf lfs.tar.gz lfs && tar -dzf lfs.tar.gz` prints nothing and exits 0;
`tar -tvzf lfs.tar.gz | grep -c '^-'` prints `0`. Replacing the target's content afterwards still gives
`tar -dzf … ; rc=0`.

**Why Blocking.** Deleting the repo does not itself remove the linked targets, so the loss is deferred,
not immediate. But the README now states the objects "have their own verified copy", and the user is
expected to note that down as the reason the copy is safe. The next time that other disk is wiped or
migrated, the only copy is gone, and the note says it was archived. The brief says a README procedure
that loses data when followed literally is the worst class; this one does, with a delay.

**Fix.** Either (a) for the store archive use `tar -czhf` and `tar -dzhf` (I verified `-h` puts
`lfs/objects/aa/bb/obj` into the archive as a regular file and that `tar -dzhf` reports `Size differs`
when the target changes), and say so in the text; or (b) keep the plain recipe but require, before
archiving a store, that `find <store> -type l` prints nothing, and otherwise archive each link's
`realpath` directory instead. Option (a) is one flag and keeps the recipe shape. Also state what to do
when the store is shared by several repos (archive once, reference it from each repo's note).

## Non-blocking

### N1 · MAJOR · `fleet/README.md:78-82` — `COPY OK` prints when the local hash is empty

**What goes wrong.** `h=$(cut -d' ' -f1 "$d/SHA256SUMS")` is empty if `SHA256SUMS` is missing or empty,
and `$(ssh … "sha256sum …")` is empty whenever the remote command fails (file missing, tool missing).
`[ "" = "" ]` is true, so the chain prints `COPY OK`.

**Trigger (reproduced).** Delete `$d/SHA256SUMS`, point the remote `sha256sum` at a name that doesn't
exist: output is a `cut` error, a `sha256sum` error, then `COPY OK: …`.

**Why not blocking.** Reaching it requires the first block to have failed (no `VERIFIED`), and the rule
on line 84-85 demands both `VERIFIED` and `COPY OK`. A literal follower is protected; a user who pastes
both blocks at once and only reads the last line is not.

**Fix.** `b=$(basename "$d"); h=$(cut -d' ' -f1 "$d/SHA256SUMS"); [ -n "$h" ] || { echo "no local hash"; false; }` as
the first link of the chain, or compare with `[ -n "$h" ] && [ "$(…)" = "$h" ]`.

### N2 · MAJOR · `bin/fleet-audit:304` — `find -L` through a link out of the store can run the host into its timeout

**What goes wrong.** With `-L`, a single shard link to a large tree is walked with every symlink inside
that tree followed too. Measured: `objects/aa -> /usr` counts 1.19 M files versus 446 k for a plain walk
(duplicates via `lib64 -> lib`, `sbin -> bin`, …), still 1.8 s on this laptop. `objects/aa -> /proc`
did not finish in 120 s (per-pid `fd`, `task`, `net` trees). On the lab box, a link into `/home` or
a data mount hits the 600 s / 1800 s budget, the whole host is reported `INCOMPLETE`, and every other
repo on it becomes unverifiable for that run.

**Why not blocking.** The failure mode is `INCOMPLETE`/`ERR`, never `ok`; nothing is silently unchecked.
It is a denial of the audit, not a false clean, and such a link is a misconfiguration.

**Fix.** Keep `-L` for correctness but bound it: `find -L "$st/objects" -maxdepth 4 -type f` (LFS stores
are exactly `objects/xx/yy/<oid>`, depth 3; anything deeper is not an object git-lfs can read, and the
`-H … -type l` pass already flags the link). Alternatively `-xdev` plus a depth cap.

### N3 · MINOR · `bin/fleet-audit:304-306` — `LFS:n` now overcounts when a link points inside the store

Store with `objects/cc -> aa` (link to a sibling shard) reports `LFS:2` for one object. Harmless for the
deletion rule (`LFS-EXTERNAL` is also set), but the number no longer means "objects on disk". Either
document that `n` is "paths readable through links" in the header comment at line 26, or count with
`find -H` and keep `find -L` only as the loop/dangling probe.

### N4 · MINOR · `fleet/README.md:79-82` — a partial rsync leaves an unnamed remote dir and the chain then refuses forever

If `rsync -a` dies mid-way, `archives/$b` exists on the remote with a partial tar.gz, and every re-run
stops at `test ! -e`. The README does not say what to do; the obvious reflex is to `rm -rf` the remote
dir by hand, which is the step the refusal exists to prevent. Say: re-run the whole recipe (a new
`mktemp` name), and only remove a remote dir whose `SOURCE` file names this host and whose hash does
not match any local `SHA256SUMS`.

### N5 · MINOR · `fleet/README.md:36-38` — the store-archive text doesn't say where the store is

`lfs.storage` is in the repo config and may be relative to `.git`, `~`-prefixed, or absolute; a linked
worktree's store is in the main repo's `.git/lfs`. The user has to rediscover what `count_lfs` already
computed. Add one line: `git -C <repo> config lfs.storage` (empty means `.git/lfs`), and for a worktree
`git rev-parse --git-common-dir`.

### N6 · NIT · `bin/fleet-audit:455` — `mktemp -d` failure exits 2

The header defines 2 as "usage/config error". A full `/tmp` is neither; it is "no host audited", which
is 1 by the header's own definition. Harmless (non-zero either way) but the exit-code contract is quoted
in the README.

### N7 · NIT · `fleet/README.md:84-85` — remote-shell claim

"The remote commands also work when the other host's shell is fish" is true for `&&`, `test`, and
`sha256sum` under `fish -c` (verified here). It assumes the ssh-session menu that `fleet-onboard`
installs stays silent for non-interactive commands; the author says this was tested against the real
lab host, and I could not test that path. Not a finding, noted for the record.

## Checked and found correct

- **`a && b && c && echo OK || echo FAIL` precedence** (`README.md:79-82`): any failing link, including
  the hash comparison, falls through to `NOT verified`; verified with five runs.
- **Refusal on an existing remote name** (`README.md:79`): the `test ! -e` runs before `rsync`, so a
  pre-existing archive with its own `SHA256SUMS` is never used as evidence. The same guard makes the
  `<other-host>` = this machine case refuse rather than self-verify.
- **Hash comparison uses the local hash** (`README.md:81`): a tampered or different remote tar.gz, and
  a remote with no `sha256sum`, both fail. Only `SOURCE` and `SHA256SUMS` themselves are not verified on
  the remote; they are not evidence, so that is fine.
- **Dangling link inside the store**: GNU find 4.11 `-L` returns 0 for a dangling link, so it does not
  become `ERR`; it becomes `LFS-EXTERNAL` via the `-H -type l` pass. Either way the row is never `ok`.
- **Loops**: `find -L` returns non-zero on "File system loop detected", so both a self-loop and a loop
  through the repo's parent end as `ERR`.
- **Containment check still runs with links only** (`bin/fleet-audit:311-315`): `n=0` plus a link enters
  the block and sets `lfsext=1` before the `realpath` test.
- **`find -H … -type l` sees a symlinked shard without descending into it**, so a link inside a linked
  shard is covered by the shard link itself.
- **Submodules and linked worktrees** pass `"$repo"` as `top`, so their stores (in `.git/modules/…` or
  the main `.git/lfs`) are judged against the directory that would be archived. Unchanged, interacts
  correctly with the new code.
- **Driver `mktemp -d` guard** is before the `trap`, so a failed mktemp never runs `rm -rf ""`.
- **Manual UNPUSHED command** (`README.md:180-184`): an unreachable origin prints the new warning; a
  reachable origin with everything pushed prints nothing and exits 0; one extra local commit lists it.
  A `cat-file` failure mid-pipe would drop negations and list more, not fewer, commits (conservative).
- **"stop writers" / `tar -d` comment** (`README.md:67-68`) is accurate: `tar -d` cannot see files added
  after the archive.

## Bottom line

**(a) Trustworthy for deletion?** `bin/fleet-audit` is: across every link layout I could construct, a
store that would not survive a plain tar of the repo directory is flagged `LFS-EXTERNAL` or `ERR`, and I
found no path to a false `ok` or a silently unchecked entry. The README is not quite: its new
instruction for clearing `LFS-EXTERNAL` by archiving the store repeats, for the store, the exact
link-vs-content mistake the tool was just fixed to detect (B1).

**(b) Ready to merge?** Not yet. Fix B1 (`tar -czhf`/`tar -dzhf` for the store archive, or a
no-symlinks precondition): one line of README. While there, add the `[ -n "$h" ]` guard (N1) and a
depth cap on `find -L` (N2); both are one-liners. With B1 fixed I would merge.

Priority: B1 → N1 → N2 → N4 → N3/N5 → N6/N7.
