# REVIEW_REQUEST: round 8 (final pass), review of `fleet` commit 55cd5d1 in cluster-utilities

**Task: adversarial CODE REVIEW, narrowly scoped.** Round 7 cleared the scanner code (no false `ok`) and
found three problems in the README's archive/copy procedure. The author says they're fixed. Review ONLY:
1. the README's "Before deleting a copy" LFS / LFS-EXTERNAL bullets,
2. the README's archive + copy recipe (the bash block under "If any of that matters for a repo"),
3. the changed object count in `count_lfs`: a plain `find -H <store>/objects -type f` at any depth,
   no link following, with any symlink below objects/ handled by the existing LFS-EXTERNAL check.
Try to BREAK each fix with concrete inputs, and look for bugs these changes introduced. Do not re-review
anything else. Do not propose new features. **If both reviewers report no Blocking findings, the branch is
merged into main.** The owner has decided in advance: if the copy step has another blocking hole, it gets
replaced by something simpler rather than patched again. So also say whether you think the current copy
step is a sound design.

## Context
cluster-utilities: scripts for a small fleet over Tailscale (a laptop and lab workstations, all CachyOS,
fish login shell everywhere, so the bash block runs under `bash`; remote commands run under the remote's
fish). `bin/fleet-audit` output plus the `fleet/README.md` procedures decide which copies are SAFE TO
DELETE. A README procedure that loses data when followed literally, or reports a verified backup that
isn't one, is the worst bug class.

## Round-7 findings and the claimed fixes, to verify
- G1 "Archive the external LFS store itself" produced an archive holding only symlinks. **Claimed:** the
  option is removed. LFS-EXTERNAL now simply means keep the copy.
- G2 `COPY OK` printed when there was no local hash (`"" = ""`). **Claimed:** the copy step first
  requires `h` to match `^[0-9a-f]{64}$`.
- G3 The remote existence test wasn't atomic. **Claimed:** the remote dir is reserved with a plain
  `mkdir` (fails if it exists), then `rsync "$d/" host:archives/$b/`, then the remote file's sha256 is
  compared with the local hash. Tested by the author against a real fish remote: a fresh name gives COPY
  OK; an existing name is refused and the old archive left untouched; no local hash is refused.
- Scanner: the LFS object count no longer follows links (round 7 had `find -L`; an interim `-maxdepth 4`
  was dropped because it could miss deeper regular files). Any symlink below objects/, dangling or
  looping included, sets LFS-EXTERNAL.

## Deliverable
Write `reviews/55cd5d1/review_<model-slug>.md` containing:
- A table: G1/G2/G3/scanner → FIXED / PARTIAL / BROKEN, with the input you tried.
- `## Blocking` and `## Non-blocking` sections (each ranked most severe first). Blocking = data loss or a
  false "verified backup" / false clean when the README is followed literally. Say why each
  non-blocking item doesn't block. Each finding gives the file:line, a CONCRETE trigger, and a fix.
- Can the new count produce a false clean (a store whose content is neither counted as LFS:n nor
  flagged LFS-EXTERNAL / ERR)?
- Bottom line: ready to merge into main? Is the copy step a sound design, or should it be replaced?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Diff since round 7 (e5acf77..55cd5d1)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index c5123ff..f9036bd 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -300,9 +300,11 @@ count_lfs() {
   absent "$st/objects" && return 0
   [ -e "$st/objects" ] || return 1
   local tf frc links; tf=$(tmpf)
-  # -L: count what git-lfs can actually read, through symlinked shard dirs or objects too. A dangling
-  # or looping link makes find fail -> ERR.
-  find -L "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
+  # Count the regular files physically in the store, at any depth. Links are NOT followed: any symlink
+  # below objects/ (a shard or object moved elsewhere, dangling or looping) is caught by the check below
+  # and makes the row LFS-EXTERNAL, a stop flag, so content behind links is never needed in the count.
+  # -H only follows objects itself if it's a link (its target is checked by the containment test).
+  find -H "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
   n=$(wc -c <"$tf"); rm -f "$tf"
   [ "$frc" = 0 ] || return 1
   # any symlink INSIDE the store (a shard dir or object moved to another disk): tar archives the link,
diff --git a/fleet/README.md b/fleet/README.md
index 3c2de51..cb2659f 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -30,15 +30,14 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
      - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
        Git can't vouch for it.
      - `LFS:n`: **keep the copy, or archive the whole repo directory first** with the recipe below,
-       but only if the row has **no `LFS-EXTERNAL`**. That flag means the LFS store lives outside the
-       repo directory (an absolute `lfs.storage`, a symlinked `objects` dir, or a linked worktree using
-       the main repo's store, or symlinked shards/objects inside the store), so the archive wouldn't
-       contain it: keep the copy. Or also archive the store itself with the same recipe (`cd` to its
-       parent, archive the store dir). The objects then have their own verified copy, which you note
-       down next to the repo's archive. A successful
-       `git lfs push --all` is NOT enough either: `lfs.url` can send the upload somewhere other than the
-       forge, and it skips LFS objects no commit points to (e.g. a file staged and then reset). Both
-       still exit 0. Don't empty the store to make the flag go away.
+       but only if the row has **no `LFS-EXTERNAL`**.
+     - `LFS-EXTERNAL`: **keep the copy.** The LFS store, or part of it, lives outside the repo directory
+       (an absolute `lfs.storage`, a symlinked `objects` dir or shards/objects inside it, or a linked
+       worktree using the main repo's store), so an archive of the repo wouldn't contain it. Archiving the
+       store separately doesn't help either: tar stores symlinks as links, and the checks still pass.
+     - For either flag, a successful `git lfs push --all` is NOT enough: `lfs.url` can send the upload
+       somewhere other than the forge, and it skips LFS objects no commit points to (e.g. a file staged
+       and then reset). Both still exit 0. Don't empty the store to make a flag go away.
      - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
        (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
      - `REPLACE:n`: `git replace -l` lists the replaced objects. If a replacement holds content you
@@ -72,17 +71,22 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    printf 'host=%s\nsource=%s\n' "$(hostname)" "$PWD/<repo>" > "$d/SOURCE"
    tar -czf "$d/<repo>.tar.gz" <repo> && tar -dzf "$d/<repo>.tar.gz" \
      && (cd "$d" && sha256sum <repo>.tar.gz > SHA256SUMS) && echo "VERIFIED: $d"
-   # copy that directory to ANOTHER machine. Refuse if the name already exists there (the random suffix
-   # is only unique on THIS machine), then check the bytes that arrived against THIS machine's hash.
-   # A copied SHA256SUMS would only prove the remote is consistent with itself.
-   b=$(basename "$d"); h=$(cut -d' ' -f1 "$d/SHA256SUMS")
-   ssh <other-host> "mkdir -p archives && test ! -e archives/$b" \
-     && rsync -a "$d" <other-host>:archives/ \
+   # copy that directory to ANOTHER machine:
+   #  - only if the step above produced a real hash (otherwise two empty strings would "match")
+   #  - reserve the name there with a plain mkdir, which fails if it already exists (the random suffix
+   #    is only unique on THIS machine), so nothing there is ever overwritten
+   #  - check the bytes that arrived against THIS machine's hash, not a copied SHA256SUMS
+   b=$(basename "$d"); h=$(cut -d' ' -f1 "$d/SHA256SUMS" 2>/dev/null)
+   [[ "$h" =~ ^[0-9a-f]{64}$ ]] \
+     && ssh <other-host> "mkdir -p archives && mkdir archives/$b" \
+     && rsync -a "$d/" "<other-host>:archives/$b/" \
      && [ "$(ssh <other-host> "sha256sum archives/$b/<repo>.tar.gz" | cut -d' ' -f1)" = "$h" ] \
      && echo "COPY OK: <other-host>:archives/$b" || echo "!! copy NOT verified: don't delete"
    ```
    `tar -d` compares the archive with the files on disk. Delete the repo only after **both**
-   `VERIFIED` and `COPY OK`. (The remote commands also work when the other host's shell is fish.)
+   `VERIFIED` and `COPY OK`. The quoted remote commands also run under fish (`&&` needs fish 3+, which
+   CachyOS has). If the copy step fails after the remote `mkdir` succeeded (e.g. rsync was
+   interrupted), check `archives/<that dir>` on the other host, remove it, and run the copy step again.
    `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
    objects (LFS, annex).
 
```
