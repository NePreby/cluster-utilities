# REVIEW_REQUEST: round 7, review of the fixes in `fleet` (commit e5acf77) in cluster-utilities

**Task: adversarial CODE REVIEW, scoped to what changed since round 6 (commit 55ebe14).** Round 6 found
the two problems below; the author says they're fixed. (1) Try to BREAK each fix with concrete inputs.
(2) Find bugs these changes introduced. Do not re-review unchanged code, except where a change interacts
with it. Do not propose new features. **If both reviewers report no Blocking findings, the branch will be
merged into main.**

## Context
cluster-utilities: scripts for a small fleet over Tailscale (a laptop, a lab workstation with ssh alias
`lab`, and 3 more CachyOS lab boxes to come). The login shell is fish on every host; the YAML is read
with awk. `bin/fleet-audit` output decides which copies are SAFE TO DELETE under the rule in
`fleet/README.md` ("Before deleting a copy"), and the README procedures are part of that rule. A false
"clean", a silently unchecked host or entry, or a README procedure that loses data when followed literally
is the worst bug class. `bin/fleet-onboard` is out of scope (unchanged; cleared by every round).

Read in full: `bin/fleet-audit` (especially `count_lfs`) and `fleet/README.md` (the "Before deleting a
copy" section and the archive recipe). The diff since round 6 is inlined at the end.

## Round-6 findings and the claimed fixes, to verify
- F1 Symlinks BELOW `<store>/objects` (symlinked shard dirs or object files) were invisible: an
  all-linked store showed no LFS flag (false clean); a mixed store showed `LFS:n` without
  `LFS-EXTERNAL`, so the README allowed a tar archive that holds only the links. **Claimed:** objects are
  counted with `find -L` (a looping link makes find fail → ERR); any symlink inside the store
  (`find -H … -mindepth 1 -type l`) sets LFS-EXTERNAL; the containment check also runs when the store
  holds only links.
- F2 The archive copy check could approve a different archive: the mktemp name is unique only locally,
  and `rsync --ignore-existing` kept a pre-existing remote archive plus its own SHA256SUMS.
  **Claimed:** refuse if the dir exists on the other host; compare the hash of the bytes that arrived
  with THIS machine's hash. Tested by the author against a real fish-shell remote.
- Also changed: the README's bash-syntax note, "stop writers" rationale, how to resolve LFS-EXTERNAL
  (archive the external store itself), the manual UNPUSHED command's failure message, and driver
  `mktemp -d` failure → exit 2.

## Also look for
- `find -L` side effects: following links out of the store into huge trees (performance only), a
  dangling link inside the store (does it end as ERR or LFS-EXTERNAL, and either way not `ok`?), races.
- The new copy chain followed literally: the `a && b && c || echo` precedence, ssh/rsync to a fish
  remote, a remote without sha256sum, partial rsync, `<other-host>` being the same machine.
- The new "archive the external store too" instruction: can following it still lose data?

## Deliverable
Write `reviews/e5acf77/review_<model-slug>.md` containing:
- A table: round-6 item → FIXED / PARTIAL / BROKEN, with the input you tried.
- Findings in TWO separate sections, each ranked most severe first:
  - `## Blocking`: must be fixed before merge or use: data loss, or a false "clean" / silently
    unchecked host or entry under the README rule. Say which.
  - `## Non-blocking`: safe to merge without it. Say in a line why it doesn't block.
  If a finding is borderline, put it under Blocking and say why. Each finding gives the file:line, what
  goes wrong, a CONCRETE trigger, and a suggested fix. Severity within a section: BLOCKER (Blocking
  only) · MAJOR · MINOR · NIT.
- What you checked and found correct.
- Bottom line: (a) fleet-audit + README trustworthy for deletion? (b) ready to merge into main?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Diff since round 6 (55ebe14..e5acf77, code + README)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index 90ebf13..c5123ff 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -25,8 +25,9 @@
 #   HIDDENIDX:n  files marked assume-unchanged / skip-worktree (on disk): status can't see their edits
 #   LFS:n        Git LFS file contents stored locally (honours lfs.storage); the server may not have
 #                them all. Re-running doesn't clear it: see fleet/README.md
-#   LFS-EXTERNAL that LFS store lives outside this repo's directory (absolute lfs.storage, symlinked
-#                objects, linked worktree), so archiving the repo dir wouldn't include it
+#   LFS-EXTERNAL that LFS store, or part of it, lives outside this repo's directory (absolute
+#                lfs.storage, a symlinked objects dir, symlinked shards/objects inside it, a linked
+#                worktree), so archiving the repo dir wouldn't include it
 #   REPLACE:n    refs/replace/* entries; a replaced blob holds content no commit count sees
 #   SUBMODULE-NOBACKUP:path   a submodule with no forge remote
 #   FETCHFAIL    --fetch of a forge remote failed, so UNPUSHED/BEHIND come from stale refs
@@ -298,13 +299,18 @@ count_lfs() {
   # no store only on a definite ENOENT; EACCES on $st (or above) or a dangling objects symlink -> ERR
   absent "$st/objects" && return 0
   [ -e "$st/objects" ] || return 1
-  local tf frc; tf=$(tmpf)
-  # -H: follow objects itself if it's a symlink (a plain find would list the link, not the store)
-  find -H "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
+  local tf frc links; tf=$(tmpf)
+  # -L: count what git-lfs can actually read, through symlinked shard dirs or objects too. A dangling
+  # or looping link makes find fail -> ERR.
+  find -L "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
   n=$(wc -c <"$tf"); rm -f "$tf"
   [ "$frc" = 0 ] || return 1
+  # any symlink INSIDE the store (a shard dir or object moved to another disk): tar archives the link,
+  # not the content, so treat it like an external store
+  links=$(find -H "$st/objects" -mindepth 1 -type l -print -quit 2>/dev/null) || return 1
   lfs=$((lfs+n))
-  if [ "$n" -gt 0 ]; then
+  if [ "$n" -gt 0 ] || [ -n "$links" ]; then
+    [ -n "$links" ] && lfsext=1
     real=$(realpath -e -- "$st/objects") || return 1
     top=$(realpath -e -- "$2") || return 1
     [[ "$real/" == "$top/"* ]] || lfsext=1
@@ -446,7 +452,8 @@ done
 echo '#DONE'
 EOF
 
-tmp=$(mktemp -d); trap 'rm -rf "$tmp"' EXIT
+tmp=$(mktemp -d) || { echo "fleet-audit: mktemp -d failed" >&2; exit 2; }
+trap 'rm -rf "$tmp"' EXIT
 tmo=600; [ "$fetch" = 1 ] && tmo=1800; tmo=${FLEET_AUDIT_TIMEOUT:-$tmo}   # per-host scan timeout (s)
 ran=()
 while IFS="$US" read -r id ssh roots; do
diff --git a/fleet/README.md b/fleet/README.md
index efaebd0..3c2de51 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -32,7 +32,10 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
      - `LFS:n`: **keep the copy, or archive the whole repo directory first** with the recipe below,
        but only if the row has **no `LFS-EXTERNAL`**. That flag means the LFS store lives outside the
        repo directory (an absolute `lfs.storage`, a symlinked `objects` dir, or a linked worktree using
-       the main repo's store), so the archive wouldn't contain it: keep the copy. A successful
+       the main repo's store, or symlinked shards/objects inside the store), so the archive wouldn't
+       contain it: keep the copy. Or also archive the store itself with the same recipe (`cd` to its
+       parent, archive the store dir). The objects then have their own verified copy, which you note
+       down next to the repo's archive. A successful
        `git lfs push --all` is NOT enough either: `lfs.url` can send the upload somewhere other than the
        forge, and it skips LFS objects no commit points to (e.g. a file staged and then reset). Both
        still exit 0. Don't empty the store to make the flag go away.
@@ -59,22 +62,27 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    If any of that matters for a repo, archive the **whole repo directory** before deleting it,
    `.git` included (that's where the reflog, LFS store and annex objects live). Each archive goes in its
    own new directory, so archiving `~/code/x` and `~/Projects/x` on the same day can't overwrite each
-   other:
+   other. **Bash syntax**: your login shell is fish, so run `bash` first.
    ```bash
-   # 0. stop anything still writing to the repo (jobs, editors, agents)
+   # 0. stop anything still writing to the repo (jobs, editors, agents). tar -d below catches files that
+   #    CHANGE after the archive is made, but not files ADDED afterwards.
    cd <parent-of-repo>                       # not inside the repo: tar would read its own output
    mkdir -p ~/archives                       # outside every fleet root (or fleet-audit flags it LOOSE)
    d=$(mktemp -d ~/archives/"$(hostname)-<repo>-$(date +%F)-XXXXXX")
    printf 'host=%s\nsource=%s\n' "$(hostname)" "$PWD/<repo>" > "$d/SOURCE"
    tar -czf "$d/<repo>.tar.gz" <repo> && tar -dzf "$d/<repo>.tar.gz" \
      && (cd "$d" && sha256sum <repo>.tar.gz > SHA256SUMS) && echo "VERIFIED: $d"
-   # copy that directory to ANOTHER machine (its name is unique; --ignore-existing never replaces
-   # anything there) and check the checksum on arrival
-   rsync -a --ignore-existing "$d" <other-host>:archives/ \
-     && ssh <other-host> "cd archives/$(basename "$d") && sha256sum -c SHA256SUMS"
+   # copy that directory to ANOTHER machine. Refuse if the name already exists there (the random suffix
+   # is only unique on THIS machine), then check the bytes that arrived against THIS machine's hash.
+   # A copied SHA256SUMS would only prove the remote is consistent with itself.
+   b=$(basename "$d"); h=$(cut -d' ' -f1 "$d/SHA256SUMS")
+   ssh <other-host> "mkdir -p archives && test ! -e archives/$b" \
+     && rsync -a "$d" <other-host>:archives/ \
+     && [ "$(ssh <other-host> "sha256sum archives/$b/<repo>.tar.gz" | cut -d' ' -f1)" = "$h" ] \
+     && echo "COPY OK: <other-host>:archives/$b" || echo "!! copy NOT verified: don't delete"
    ```
-   `tar -d` compares the archive with the files on disk. Delete the repo only after **both** `VERIFIED`
-   and `<repo>.tar.gz: OK` from the other machine.
+   `tar -d` compares the archive with the files on disk. Delete the repo only after **both**
+   `VERIFIED` and `COPY OK`. (The remote commands also work when the other host's shell is fish.)
    `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
    objects (LFS, annex).
 
@@ -172,7 +180,8 @@ because cached tracking refs can be stale:
 git fetch --prune origin && adv=$(git ls-remote origin) && printf '%s\n' "$adv" \
   | awk -F'\t' 'NF {print $1}' | git cat-file --batch-check='%(objectname) %(objecttype)' \
   | awk '$2 ~ /^(commit|tag|tree|blob)$/ {print "^" $1}' \
-  | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin
+  | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin \
+  || echo "!! couldn't ask origin (fetch or ls-remote failed): no answer, NOT 'nothing unpushed'"
 ```
 
 Keep `hosts.yaml` / `projects.yaml` in their flat format, because they are read with awk. Inline
```
