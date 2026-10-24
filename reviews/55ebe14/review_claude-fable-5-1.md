# Review of commit 55ebe14 (round 6) — `bin/fleet-audit` + `fleet/README.md`

**Reviewer:** Claude Fable 5.1 (model id `claude-fable-5-1`).
**Scope:** the diff c3eeb4d..55ebe14 and its interactions with unchanged code. `bin/fleet-onboard` untouched except for
one grep (the fish ssh menu's `status is-interactive` guard, line 126) because the new recipe runs `ssh`/`rsync` into a
fleet box.

**Method.** I read `bin/fleet-audit` and `fleet/README.md` in full, then built 16 fixture repos under a scratch root with
a two-host `hosts.yaml` (`ssh: local`) and ran the real script against them (git 2.56, git-lfs 3.8, GNU tar 1.35,
rsync 3.5, fish 4.9). The archive recipe was run literally under bash and under fish. Exit codes quoted below are
observed, not assumed.

## Round-5 items

| Item | Verdict | Inputs tried (all run) |
|---|---|---|
| E1 `LFS-EXTERNAL` for stores outside the repo dir | **PARTIAL** | Absolute `lfs.storage` → flagged. `objects` is a symlink out → flagged. `~/…` storage → flagged. Relative `../../../ext` storage → flagged. Linked worktree audited as its own entry (`G-wt`) → flagged. `--separate-git-dir` and `.git`-as-symlink → flagged. Submodule store in `.git/modules/sub` → correctly NOT flagged. Store under a symlinked root → correctly NOT flagged. Empty `lfs.storage` → counted from the default dir, matching `git lfs env`. **Broken:** a store whose shard dirs or object files are symlinks (`objects/ab -> /mnt/...`): no `LFS-EXTERNAL`, and when every shard is a symlink, no `LFS` flag at all (see B1). |
| E2 archive name collisions / verification | **FIXED** | Ran the block literally: unique `mktemp -d` dir, `SOURCE`, `SHA256SUMS` with a relative name, `VERIFIED: <dir>` only after `tar -d` exit 0. `tar -d` exits 1 on a changed file (chain stops). `rsync -a --ignore-existing <dir> dest/` into a dest that doesn't exist yet still lands as `dest/<dir>/…`. Empty `$d` (fish) → rsync refuses ("Empty source arg"). `$(basename "$d")` expands locally as intended. |
| Empty `lfs.storage` → default | FIXED | `git config lfs.storage ""` + objects in `.git/lfs/objects` → `LFS:1`; `git lfs env` reports the same dir. |
| scanner `mktemp -d` failure → non-zero | FIXED | Ran the embedded scanner with `TMPDIR=/nonexistent`: prints `scanner: mktemp -d failed`, exit 1, host reported `NOT AUDITED`. |
| README manual UNPUSHED command is an `&&` chain | FIXED | Pointed `origin` at `https://127.0.0.1:9/`: stdout empty, fatal on stderr. Shell precedence is `A && B && (C \| D \| E \| F)` as intended. |
| "stop writers first", timeout comment | FIXED | Text only. Comment matches bash's trap-after-current-command behaviour. |

## Blocking

### B1 · BLOCKER · symlinks *inside* the LFS store are invisible: false "clean" (no `LFS` flag) or a silently incomplete archive
**Where:** `bin/fleet-audit:298-310` (`count_lfs`: `find -H "$st/objects" -type f` and the `realpath -e "$st/objects"` check).

**What goes wrong.** `find -H` follows only the command-line argument. Anything reached through a symlink *below*
`objects/` is not a `-type f` and is not counted, and the external check only resolves `objects` itself. Two concrete
fixtures, both run through the real script:

| fixture | layout | row shows |
|---|---|---|
| `D2` | `.git/lfs/objects/ab -> /tmp/ext/D2-store/objects/ab` (one object behind it) | **no `LFS`, no `LFS-EXTERNAL`** (row is otherwise `ok`) |
| `D` | one real object + `objects/ef -> /mnt/…` + `objects/ab/cd/<oid> -> /mnt/…` | `LFS:1`, **no `LFS-EXTERNAL`** |

`find -L` on the same `D2` store returns 1, so git-lfs itself can read that object (it just opens the path).
Symlinking shard dirs or individual objects of a store onto a bigger disk is a real way people move LFS data without
touching `lfs.storage`.

**Why it's Blocking under the README rule.** For `D2` the row carries no LFS stop flag at all, so a repo with every
LFS object parked behind a shard symlink is deletable as `ok`: a false clean. For `D` the row says `LFS:1` with no
`LFS-EXTERNAL`, so `fleet/README.md:32-35` explicitly allows the tar route; `tar -czf` without `-h` stores the symlinks,
`tar -d` compares symlink targets and passes, `sha256sum -c` passes on the other machine, and the user deletes a repo whose
"verified" archive contains none of the externally stored LFS content. That is exactly the E1 failure, one directory
level down.

**Fix (small).** Count through symlinks and treat *any* symlink inside the store as external, since the archive won't
follow it either:
```bash
find -L "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?     # count what git-lfs can actually read
...
if [ "$n" -gt 0 ]; then
  real=$(realpath -e -- "$st/objects") || return 1
  top=$(realpath -e -- "$2") || return 1
  [[ "$real/" == "$top/"* ]] || lfsext=1
  # a symlinked shard or object: tar would archive the link, not the content
  [ -n "$(find "$st/objects" -type l -print -quit 2>/dev/null)" ] && lfsext=1
fi
```
`find -L` with a dangling link inside prints an error and exits non-zero, which the existing `[ "$frc" = 0 ] || return 1`
already turns into `ERR`. Add the `D`/`D2` layouts to whatever fixture set the earlier rounds used.

## Non-blocking

### N1 · MINOR · the archive block has no "bash syntax" note, unlike the UNPUSHED block
**Where:** `fleet/README.md:63-75` vs the note added at `fleet/README.md:170`.
**Tried:** pasted the block literally into fish 4.9, both as a file and as one interactive buffer. Fish rejects the
whole thing at line 67 (`Unsupported use of '='`), exit 127, nothing created. Even after hand-converting that line to
`set d (…)`, fish refuses line 70 (`command substitutions not allowed in command position`). With `$d` unset, the rsync
line fails with "Empty source arg". No path prints `VERIFIED` or `OK`, so this can't cause a bad deletion. It just wastes
the user's time: add the same one-line note (`# bash syntax: in fish, run with bash`) as the UNPUSHED block has.
**Doesn't block because** every fish failure is loud and happens before anything is written.

### N2 · MINOR · `tar -d` cannot see files *added* after `tar -c`
**Where:** `fleet/README.md:69-70,76`. Verified: add a file after archiving, `tar -dzf` still exits 0.
Step 0 ("stop anything still writing") is the mitigation, and the text "no differences" slightly oversells `-d`. One
clause ("`tar -d` checks the files in the archive, not files created since") would make the rule accurate.
**Doesn't block because** the recipe already requires writers to be stopped first, and this is pre-existing semantics.

### N3 · MINOR (pre-existing, interacts with E1) · separate git dirs show a spurious `WORKTREES:1 ERR`
**Where:** `bin/fleet-audit:352-358`. For `git init --separate-git-dir` and for a `.git` that is a symlink,
`git worktree list --porcelain` reports the git dir path as the main worktree (observed: `worktree /tmp/.../ext/H-git`),
so the loop counts it as linked and `count_status` on it fails with "must be run in a work tree". The new
`LFS-EXTERNAL` is correct on these repos, and the extra flags are stop flags, so this is a false *positive* only.
**Doesn't block because** it can only prevent a deletion, never allow one.

### N4 · NIT · manual UNPUSHED command: "server unreachable" and "everything pushed" both print nothing on stdout
**Where:** `fleet/README.md:172-175`. The fatal goes to stderr, which is visible interactively, but a `|| echo
'inconclusive: remote unreachable'` at the end would make the two outcomes unmistakable.

### N5 · NIT · the outer `tmp=$(mktemp -d)` is still unchecked
**Where:** `bin/fleet-audit:465`. With `TMPDIR=/nonexistent` the run spews `integer expected` / `/me.err: No such
file` errors but still exits 1 and reports `NOT AUDITED`. Same `|| { …; exit 2; }` guard as the scanner would be tidier.

### N6 · NIT · `LFS-EXTERNAL` has no documented way out
`fleet/README.md:33-35` says "keep the copy" and nothing else. That is the safe answer and I'm not proposing a feature,
but a user with an absolute `lfs.storage` on a box they want to retire will have to invent a procedure. A pointer such as
"archive the store directory printed by `git lfs env` (LocalMediaDir) alongside the repo" would keep them inside the
README.

## Checked and found correct
- `[[ "$real/" == "$top/"* ]]`: the trailing slash prevents `x-lfs/` matching `x/`; right-hand side is quoted so glob
  characters in a path stay literal.
- `realpath -e` on both sides handles a symlinked fleet root, `..` in `lfs.storage`, a symlinked `objects` dir, a
  symlinked `.git`, and `--separate-git-dir` (all exercised).
- Submodule stores (`.git/modules/<name>/lfs`) resolve under the superproject and are not flagged. Linked worktrees
  audited as entries are flagged; the main repo gets `WORKTREES:n` so deleting it is stopped.
- Empty `lfs.storage` falls back to the default dir and agrees with `git lfs env`. `~user/…` (fixture `K2`): git-lfs
  itself does not expand it (its `LocalMediaDir` is the literal `.git/~yorguin/…`), so the scanner's `~*` → `$HOME…`
  mapping and git-lfs both see no store; no disagreement in practice.
- `lfsext` is only consulted when `n > 0`; `return 0` at the end restores the old success path, and `|| err=1` callers
  still get `ERR` on `realpath -e` failure.
- Scanner `mktemp -d` failure → exit 1 → `NOT AUDITED`; trap/`exit 143` unchanged. `#DONE` still emitted on a normal run.
- Archive recipe under bash: unique dir name, `SOURCE` has host and absolute source path, `SHA256SUMS` uses a relative
  filename so `sha256sum -c` works after rsync, `VERIFIED` only after `tar -d` exit 0, `tar -d` exit 1 on a changed
  file (chain stops before `VERIFIED`). `rsync -a` writes to a temp name and renames, so an interrupted copy leaves no
  same-named partial for `--ignore-existing` to skip over; a stale same-named file would still fail `sha256sum -c`.
  `rsync` into a not-yet-existing `archives/` on the other host produces `archives/<dir>/…`, not a flattened dir.
- `ssh <host> "cd … && sha256sum -c …"` is fine with a fish login shell (fish supports `&&`), and the fleet's ssh session
  menu is guarded by `status is-interactive` (`bin/fleet-onboard:126`), so neither rsync's server process nor the check
  command is hijacked.
- README stop-flag list now includes `LFS-EXTERNAL`; the LFS paragraph makes the tar route conditional on its absence.

## Bottom line
**(a) Trustworthy for deletion?** Not yet. For every layout the round-5 reviewers named, the flag now fires. But the one
layout the brief asked about explicitly, a store whose objects are partly or wholly behind symlinks inside `objects/`,
still produces either no `LFS` flag at all (every shard symlinked) or `LFS:n` without `LFS-EXTERNAL`, and the README
then blesses an archive that contains only symlinks. That is a false clean in the first case and a verified-but-empty
backup in the second.

**(b) Ready to merge?** No. One Blocking finding (B1), with a two-line fix in `count_lfs` (`find -L` for the count, any
symlink inside the store → `LFS-EXTERNAL`). Everything else is cosmetic. After B1 is fixed and the `D`/`D2` layouts are
in the fixtures, I would merge.

Priority order: B1 → N1 → N2 → N3/N4/N5/N6 at leisure.
