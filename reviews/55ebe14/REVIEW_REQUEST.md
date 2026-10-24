# REVIEW_REQUEST: round 6, review of the fixes in `fleet` (commit 55ebe14) in cluster-utilities

**Task: adversarial CODE REVIEW, scoped to what changed since round 5 (commit c3eeb4d).** Round 5 found
the two problems below; the author says they're fixed. (1) Try to BREAK each fix with concrete inputs.
(2) Find bugs these changes introduced. Do not re-review unchanged code, except where a change interacts
with it. Do not propose new features. **If both reviewers report no Blocking findings, the branch will be
merged into main.**

## Context
cluster-utilities: scripts for a small fleet over Tailscale (a laptop, a lab workstation with ssh alias
`lab`, and 3 more CachyOS lab boxes to come). The login shell is fish; the YAML is read with awk. `bin/fleet-audit`
output decides which copies are SAFE TO DELETE under the rule in `fleet/README.md` ("Before deleting a
copy"), and the README procedures are part of that rule. A false "clean", a silently unchecked host or
entry, or a README procedure that loses data when followed literally is the worst bug class.
`bin/fleet-onboard` is out of scope (unchanged; cleared by every previous round).

Read in full: `bin/fleet-audit` and `fleet/README.md`. The diff since round 5 is inlined at the end.

## Round-5 findings and the claimed fixes, to verify
- E1 The tar recipe could "clear" LFS while the store lived outside the repo dir (absolute
  `lfs.storage`, symlinked `objects`), and tar kept only the symlink. **Claimed:** a new stop flag,
  `LFS-EXTERNAL`, when the resolved store (`realpath -e` of `<store>/objects`) isn't under the
  `realpath` of the repo dir being audited (also covers a linked worktree whose store is in the main repo).
  The README allows the tar route for LFS only without that flag.
- E2 Archive name collisions (`~/archives/<repo>-<date>.tar.gz` for `~/code/x` and `~/Projects/x`).
  **Claimed:** a fresh `mktemp -d` dir per archive (host-repo-date-random) with SOURCE and SHA256SUMS,
  copied to another machine with `rsync --ignore-existing` and verified with `sha256sum -c` there.
  Delete only after VERIFIED + OK.
- Also changed: an empty `lfs.storage` falls back to the default; `mktemp -d` failure → non-zero (host
  NOT AUDITED); the README's manual UNPUSHED command is an `&&` chain so a dead remote prints nothing;
  "stop writers first"; the timeout comment was corrected.

## Also look for
- `LFS-EXTERNAL` edge cases: stores in submodules (`.git/modules/x`), a repo dir that is itself a
  symlink, a store whose objects are partly symlinks, repos where `.git` is a gitfile.
- Whether the archive recipe, followed literally (including from fish, where the bash block must be run
  under bash), can still lose data or verify the wrong thing.

## Deliverable
Write `reviews/55ebe14/review_<model-slug>.md` containing:
- A table: round-5 item → FIXED / PARTIAL / BROKEN, with the input you tried.
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

## Diff since round 5 (c3eeb4d..55ebe14, code + README)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index 8cdb715..90ebf13 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -25,6 +25,8 @@
 #   HIDDENIDX:n  files marked assume-unchanged / skip-worktree (on disk): status can't see their edits
 #   LFS:n        Git LFS file contents stored locally (honours lfs.storage); the server may not have
 #                them all. Re-running doesn't clear it: see fleet/README.md
+#   LFS-EXTERNAL that LFS store lives outside this repo's directory (absolute lfs.storage, symlinked
+#                objects, linked worktree), so archiving the repo dir wouldn't include it
 #   REPLACE:n    refs/replace/* entries; a replaced blob holds content no commit count sees
 #   SUBMODULE-NOBACKUP:path   a submodule with no forge remote
 #   FETCHFAIL    --fetch of a forge remote failed, so UNPUSHED/BEHIND come from stale refs
@@ -134,8 +136,12 @@ done <<<"$proj_tsv"
 read -r -d '' SCAN <<'EOF'
 fetch=$1; size=$2; shift 2
 set -o pipefail
-# all temp files live here and go away on exit, also when `timeout` kills the scan (TERM -> exit -> EXIT trap)
-SCANTMP=$(mktemp -d); trap 'rm -rf "$SCANTMP"' EXIT; trap 'exit 143' TERM INT HUP
+# all temp files live here and go away on exit. On a TERM (fleet-audit's per-host timeout, local hosts)
+# bash runs the trap once the current command returns, which can be up to a 90 s fetch later. Over ssh
+# the remote scan isn't signalled at all and cleans up when it finishes. Either way the host is already
+# reported NOT AUDITED/INCOMPLETE, so a late cleanup never affects the result.
+SCANTMP=$(mktemp -d) || { echo "scanner: mktemp -d failed" >&2; exit 1; }
+trap 'rm -rf "$SCANTMP"' EXIT; trap 'exit 143' TERM INT HUP
 tmpf() { mktemp -p "$SCANTMP"; }
 # 0 only if $1 definitely doesn't exist (ENOENT). A permission error on a parent, or a dangling
 # symlink, is "can't tell", never "absent". Only the END of stat's message is matched: the message
@@ -280,10 +286,14 @@ count_hidden() {
 # LFS file contents stored locally (the server may not have them if a push skipped the LFS hook).
 # Storage is <git dir>/lfs unless lfs.storage moves it (relative values are inside the git dir).
 # A store that exists but can't be fully listed is an error, never "0 objects".
+# $2 = the directory a copy of this repo would be archived from. If the resolved store isn't inside it
+# (absolute lfs.storage, a symlinked objects dir, a linked worktree whose store is in the main repo's
+# .git), lfsext=1: archiving that directory would NOT include the LFS content.
 count_lfs() {
-  local gd st n
+  local gd st n real top
   gd=$(g "$1" rev-parse --path-format=absolute --git-common-dir) || return 1
   st=$(g "$1" config --get lfs.storage) || st="lfs"
+  [ -z "$st" ] && st="lfs"   # an empty value would otherwise point at the git object store
   case "$st" in /*) ;; ~*) st="$HOME${st#\~}" ;; *) st="$gd/$st" ;; esac
   # no store only on a definite ENOENT; EACCES on $st (or above) or a dangling objects symlink -> ERR
   absent "$st/objects" && return 0
@@ -294,6 +304,12 @@ count_lfs() {
   n=$(wc -c <"$tf"); rm -f "$tf"
   [ "$frc" = 0 ] || return 1
   lfs=$((lfs+n))
+  if [ "$n" -gt 0 ]; then
+    real=$(realpath -e -- "$st/objects") || return 1
+    top=$(realpath -e -- "$2") || return 1
+    [[ "$real/" == "$top/"* ]] || lfsext=1
+  fi
+  return 0
 }
 
 # refs/replace/*: a replaced commit is caught by UNPUSHED, but a replaced BLOB (or tree) holds content
@@ -309,7 +325,7 @@ emit() { printf '%s\x1f%s\x1f%s\x1f%s\x1f%s\x1f%s\n' "${1:--}" "${2:--}" "${3:--
 
 scan_repo() {
   local root=$1 n=$2 repo=$3
-  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 hid=0 lfs=0 repl=0 fetchfail=0 rc
+  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 hid=0 lfs=0 lfsext=0 repl=0 fetchfail=0 rc
   local ignpaths=() flags=() remotes backups r u wt line s p
   remotes=$(g "$repo" remote) || err=1
   backups=$(backup_remotes "$repo") || err=1
@@ -324,7 +340,7 @@ scan_repo() {
 
   count_status "$repo" || err=1
   count_hidden "$repo" || err=1
-  count_lfs "$repo" || err=1
+  count_lfs "$repo" "$repo" || err=1
   count_replace "$repo" || err=1
   if [ -n "$backups" ]; then
     count_unpushed "$repo" "$backups"; rc=$?
@@ -351,7 +367,7 @@ scan_repo() {
     nsub=$((nsub+1))
     count_status "$p" || err=1
     count_hidden "$p" || err=1
-    count_lfs "$p" || err=1
+    count_lfs "$p" "$repo" || err=1
     count_replace "$p" || err=1
     u=$(backup_remotes "$p") || err=1
     if [ -z "$u" ]; then flags+=("SUBMODULE-NOBACKUP:${p#"$repo"/}")
@@ -373,6 +389,7 @@ scan_repo() {
   [ "$nsub" -gt 0 ] && flags+=("SUBMODULES:$nsub")
   [ "$hid" -gt 0 ] && flags+=("HIDDENIDX:$hid")
   [ "$lfs" -gt 0 ] && flags+=("LFS:$lfs")
+  [ "$lfsext" = 1 ] && flags+=(LFS-EXTERNAL)
   [ "$repl" -gt 0 ] && flags+=("REPLACE:$repl")
   [ "$fetchfail" = 1 ] && flags+=(FETCHFAIL)
   [ "$err" = 1 ] && flags+=(ERR)
diff --git a/fleet/README.md b/fleet/README.md
index 8c105ba..efaebd0 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -24,15 +24,18 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
      UNLISTED, LEGACY, EMPTY (an empty dir: `rmdir` it). `UNDECIDED` means decide the owner first.
    - **Stop flags.** Any of these means the copy holds something not saved elsewhere: NOGIT, NESTED,
      NOREMOTE, LOCALREMOTE, SUBMODULE-NOBACKUP, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, HIDDENIDX,
-     LFS, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run.
+     LFS, LFS-EXTERNAL, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then
+     re-run.
      `IGNORED` and `LFS` stay on re-runs, so they need a check you do yourself (and note down):
      - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
        Git can't vouch for it.
-     - `LFS:n`: **keep the copy, or archive the whole repo directory first** (the tar recipe below,
-       including the `tar -df` check). A successful `git lfs push --all` is NOT enough: `lfs.url` can
-       send the upload somewhere other than the forge, and it skips LFS objects no commit points to
-       (e.g. a file staged and then reset). Both still exit 0. Don't empty the store to make the flag
-       go away.
+     - `LFS:n`: **keep the copy, or archive the whole repo directory first** with the recipe below,
+       but only if the row has **no `LFS-EXTERNAL`**. That flag means the LFS store lives outside the
+       repo directory (an absolute `lfs.storage`, a symlinked `objects` dir, or a linked worktree using
+       the main repo's store), so the archive wouldn't contain it: keep the copy. A successful
+       `git lfs push --all` is NOT enough either: `lfs.url` can send the upload somewhere other than the
+       forge, and it skips LFS objects no commit points to (e.g. a file staged and then reset). Both
+       still exit 0. Don't empty the store to make the flag go away.
      - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
        (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
      - `REPLACE:n`: `git replace -l` lists the replaced objects. If a replacement holds content you
@@ -54,15 +57,24 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    - git-annex content.
 
    If any of that matters for a repo, archive the **whole repo directory** before deleting it,
-   `.git` included (that's where the reflog, LFS store and annex objects live), and check the archive
-   against the disk:
+   `.git` included (that's where the reflog, LFS store and annex objects live). Each archive goes in its
+   own new directory, so archiving `~/code/x` and `~/Projects/x` on the same day can't overwrite each
+   other:
    ```bash
-   cd <parent-of-repo>                        # not inside the repo: tar would read its own output
-   a=~/archives/<repo>-$(date +%F).tar.gz     # outside every fleet root (or fleet-audit flags it LOOSE)
-   mkdir -p ~/archives && tar -czf "$a" <repo> && tar -dzf "$a" && echo VERIFIED
+   # 0. stop anything still writing to the repo (jobs, editors, agents)
+   cd <parent-of-repo>                       # not inside the repo: tar would read its own output
+   mkdir -p ~/archives                       # outside every fleet root (or fleet-audit flags it LOOSE)
+   d=$(mktemp -d ~/archives/"$(hostname)-<repo>-$(date +%F)-XXXXXX")
+   printf 'host=%s\nsource=%s\n' "$(hostname)" "$PWD/<repo>" > "$d/SOURCE"
+   tar -czf "$d/<repo>.tar.gz" <repo> && tar -dzf "$d/<repo>.tar.gz" \
+     && (cd "$d" && sha256sum <repo>.tar.gz > SHA256SUMS) && echo "VERIFIED: $d"
+   # copy that directory to ANOTHER machine (its name is unique; --ignore-existing never replaces
+   # anything there) and check the checksum on arrival
+   rsync -a --ignore-existing "$d" <other-host>:archives/ \
+     && ssh <other-host> "cd archives/$(basename "$d") && sha256sum -c SHA256SUMS"
    ```
-   `tar -d` compares the archive with the files on disk. Only `VERIFIED` (exit 0, no differences)
-   counts. Then copy the archive somewhere that isn't this machine before deleting the repo.
+   `tar -d` compares the archive with the files on disk. Delete the repo only after **both** `VERIFIED`
+   and `<repo>.tar.gz: OK` from the other machine.
    `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
    objects (LFS, annex).
 
@@ -155,8 +167,10 @@ GitHub shows up here too. To see the actual commits before assuming work is at r
 server the same way the tool does (forge remote `origin` here). Don't use `--not --remotes=origin`,
 because cached tracking refs can be stale:
 ```bash
-git fetch --prune origin
-git ls-remote origin | awk -F'\t' 'NF {print $1}' | git cat-file --batch-check='%(objectname) %(objecttype)' \
+# bash syntax (in fish: run it inside `bash -c '…'`). The && chain stops if the server can't be reached;
+# otherwise an empty answer would make git log list ALL local history as unpushed.
+git fetch --prune origin && adv=$(git ls-remote origin) && printf '%s\n' "$adv" \
+  | awk -F'\t' 'NF {print $1}' | git cat-file --batch-check='%(objectname) %(objecttype)' \
   | awk '$2 ~ /^(commit|tag|tree|blob)$/ {print "^" $1}' \
   | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin
 ```
```
