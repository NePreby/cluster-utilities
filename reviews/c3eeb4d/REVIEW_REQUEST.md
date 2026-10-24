# REVIEW_REQUEST: round 5, review of the fixes in `fleet` (commit c3eeb4d) in cluster-utilities

**Task: adversarial CODE REVIEW, scoped to what changed since round 4 (commit 7594a87).** Round 4
found the problems listed below; the author says they're fixed. (1) Try to BREAK each fix with concrete
inputs. (2) Find bugs these changes introduced. Do not re-review unchanged code, except where a change
interacts with it. Do not propose new features. **If both reviewers report no Blocking findings, the
branch will be merged into main.**

## Context
cluster-utilities holds the owner's scripts for a small fleet of machines over Tailscale: a laptop, a lab
workstation (ssh alias `lab`), and 3 more CachyOS lab workstations about to be onboarded. All hosts run
CachyOS; the login shell is fish; the YAML is read with awk (no YAML parser on the hosts). Password ssh
login is intentionally LEFT ENABLED (do not flag it). The real remotes are GitHub plus a few Overleaf
(those give FETCHFAIL under `--fetch` because they need credentials; that's accepted). Several real repos
use Git LFS.

- `bin/fleet-audit`: its output decides which copies are SAFE TO DELETE under the rule in
  `fleet/README.md` ("Before deleting a copy"). A false "clean" (unique work reported ok, or a host or
  entry silently not checked) is the worst bug class. The README is part of that rule.
- `bin/fleet-onboard`: unchanged since round 4, where both reviewers found no blocking issue. Out of
  scope.

Read in full: `bin/fleet-audit` and `fleet/README.md`. The diff since round 4 is inlined at the end.

## Round-4 findings and the claimed fixes, to verify
- D1 An LFS store that exists but can't be traversed (EACCES on `.git/lfs` or a custom store dir), a
  dangling `objects` symlink, or a symlinked `objects` dir was counted as 0. **Claimed:** an `absent()`
  helper means "no store" only on a definite ENOENT; anything else is ERR; `find -H`.
- D2 The README let a successful `git lfs push --all` clear LFS (`lfs.url` can point elsewhere;
  unreferenced objects are skipped). **Claimed:** LFS is never cleared by a push: keep the copy or archive
  the whole repo dir (tar + `tar -d` verification) first.
- D3 The ENOENT test matched the path inside stat's message. **Claimed:** `absent()` matches only the
  message suffix `": No such file or directory"`. Also new: every root in hosts.yaml must start with `/`
  or `~` (roots are space-separated).
- N1/N2 An empty server advertisement became `^missing` (ERR); cat-file errors were ignored.
  **Claimed:** blank lines dropped, only well-formed cat-file output used, cat-file failure or stderr
  means ERR.
- N3 The README's manual UNPUSHED command used cached refs. **Claimed:** it now uses ls-remote like the tool.
- N4 `git replace -d`. N5 tar recipe (cwd, archive location outside roots, `tar -d`). N6 scanner temp
  files in one dir removed on exit including a timeout kill; awk prefix for warnings. N7 tab wording.

## Also look for
- `absent()` edge cases: non-English locales (it sets LC_ALL=C), ENOTDIR (a file where a dir is
  expected), ELOOP, `stat` exit status vs message, paths ending in the suffix text.
- The scanner's `trap 'exit 143' TERM INT HUP` plus the EXIT trap under `timeout` and under ssh; whether
  a killed scan can still print `#DONE`.
- Whether the new README instructions (LFS → keep or verified tar; tar recipe; manual command) are safe
  and correct if followed literally.

## Deliverable
Write `reviews/c3eeb4d/review_<model-slug>.md` containing:
- A table: round-4 item → FIXED / PARTIAL / BROKEN, with the input you tried.
- Findings in TWO separate sections, each ranked most severe first:
  - `## Blocking`: must be fixed before merge or use: data loss, or a false "clean" / silently
    unchecked host or entry under the README rule. Say which.
  - `## Non-blocking`: safe to merge without it. Say in a line why it doesn't block.
  If a finding is borderline, put it under Blocking and say why. Each finding gives the file:line, what
  goes wrong, a CONCRETE trigger, and a suggested fix. Severity within a section: BLOCKER (Blocking
  only) · MAJOR · MINOR · NIT.
- What you checked and found correct.
- Bottom line: (a) fleet-audit trustworthy for deletion under the README rule? (b) ready to merge into main?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Diff since round 4 (7594a87..c3eeb4d, code + README)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index 8d2346d..8cdb715 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -75,6 +75,8 @@ hosts_all=$(awk -v US="$US" '
     if (st == "") { err("host " id ": missing status:") }
     else if (st != "active" && st != "planned" && st != "retired") { err("host " id ": status must be active|planned|retired, got " st) }
     if (st == "active" && (ssh == "" || roots == "")) { err("active host " id ": needs ssh: and roots:") }
+    # roots are space-separated, so a path with a space would silently split into bogus roots
+    nr=split(roots, rs, " "); for (i=1; i<=nr; i++) if (rs[i] !~ /^(\/|~)/) err("host " id ": root " rs[i] " must start with / or ~ (roots are space-separated; paths with spaces are not supported)")
     print id US ssh US roots US st; id="" }
   function err(m) { print "hosts.yaml: " m > "/dev/stderr"; bad=1 }
   { line=$0; sub(/\r$/, "", line); sub(/[[:space:]]+#.*$/, "", line); sub(/^[[:space:]]*#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
@@ -132,6 +134,19 @@ done <<<"$proj_tsv"
 read -r -d '' SCAN <<'EOF'
 fetch=$1; size=$2; shift 2
 set -o pipefail
+# all temp files live here and go away on exit, also when `timeout` kills the scan (TERM -> exit -> EXIT trap)
+SCANTMP=$(mktemp -d); trap 'rm -rf "$SCANTMP"' EXIT; trap 'exit 143' TERM INT HUP
+tmpf() { mktemp -p "$SCANTMP"; }
+# 0 only if $1 definitely doesn't exist (ENOENT). A permission error on a parent, or a dangling
+# symlink, is "can't tell", never "absent". Only the END of stat's message is matched: the message
+# also contains the path, which could itself contain the words.
+absent() {
+  local m
+  [ -e "$1" ] && return 1
+  [ -L "$1" ] && return 1
+  m=$(LC_ALL=C stat -- "$1" 2>&1 >/dev/null)
+  [[ "$m" == *": No such file or directory" ]]
+}
 shopt -s nullglob dotglob
 # GIT_NO_REPLACE_OBJECTS: walk the real history; replace refs themselves are still counted via --all
 export GIT_TERMINAL_PROMPT=0 GIT_NO_REPLACE_OBJECTS=1
@@ -151,11 +166,11 @@ lines() { [ -z "$1" ] && echo 0 || printf '%s\n' "$1" | wc -l; }
 # copy entry (R/C in the XY code) is followed by one extra record, the source path, which is skipped.
 count_status() {
   local d=$1 tf rec p skip=0 rc=0
-  tf=$(mktemp)
+  tf=$(tmpf)
   # git only WARNS (exit 0) about dirs it can't open, e.g. a root-owned docker/sudo output dir, and the
   # untracked files inside are then missing from the counts. So any stderr output means ERR.
   g "$d" status --porcelain -z --untracked-files=normal --ignored=matching --ignore-submodules=none >"$tf" 2>"$tf.err" || rc=1
-  [ -s "$tf.err" ] && { rc=1; head -c 300 "$tf.err" | sed "s#^#$d: #" >&2; }
+  [ -s "$tf.err" ] && { rc=1; head -c 300 "$tf.err" | awk -v d="$d" '{print d ": " $0}' >&2; }
   while IFS= read -r -d '' rec; do
     if [ "$skip" = 1 ]; then skip=0; continue; fi
     case "$rec" in
@@ -219,14 +234,19 @@ fetch_backups() {
 # Refs of remotes that aren't backups (or no longer exist) never count. Returns 1 on a git error,
 # 2 if a server couldn't be asked (-> FETCHFAIL).
 count_unpushed() {
-  local d=$1 r n adv tf rc=0
-  tf=$(mktemp)
+  local d=$1 r n adv tf rc=0 bad=0
+  tf=$(tmpf)
   if [ "$fetch" = 1 ]; then
     for r in $2; do
       adv=$(timeout 60 git --no-optional-locks -C "$d" ls-remote "$r" </dev/null 2>/dev/null) || { rc=2; continue; }
-      # keep only objects we have locally (rev-list can't negate what it can't read); ^ = "not"
-      cut -f1 <<<"$adv" | sort -u | g "$d" cat-file --batch-check='%(objectname) %(objecttype)' 2>/dev/null \
-        | awk '$2 != "missing" {print "^" $1}' >>"$tf"
+      # An empty advertisement (repo created on the forge, nothing pushed) is valid: nothing to negate.
+      # Keep only objects we have locally (rev-list can't negate what it can't read); ^ = "not".
+      # cat-file errors or warnings (e.g. a corrupt object) -> ERR, never silently dropped.
+      printf '%s\n' "$adv" | awk -F'\t' 'NF && $1 != "" {print $1}' | sort -u >"$tf.in" || bad=1
+      g "$d" cat-file --batch-check='%(objectname) %(objecttype)' <"$tf.in" >"$tf.out" 2>"$tf.err" || bad=1
+      [ -s "$tf.err" ] && { bad=1; head -c 300 "$tf.err" | awk -v d="$d" '{print d ": " $0}' >&2; }
+      awk '$1 ~ /^[0-9a-f]+$/ && $2 ~ /^(commit|tag|tree|blob)$/ {print "^" $1}' "$tf.out" >>"$tf" || bad=1
+      rm -f "$tf.in" "$tf.out" "$tf.err"
     done
     n=$(g "$d" rev-list --count --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin <"$tf") || { rm -f "$tf"; return 1; }
   else
@@ -236,6 +256,7 @@ count_unpushed() {
   fi
   rm -f "$tf"
   unp=$((unp+n))
+  [ "$bad" = 1 ] && return 1
   return $rc
 }
 
@@ -244,7 +265,7 @@ count_unpushed() {
 # -z: names are never quoted, so a file with a tab or quote in its name is tested under its real name
 count_hidden() {
   local d=$1 tf rec rc=0
-  tf=$(mktemp)
+  tf=$(tmpf)
   g "$d" ls-files -v -z >"$tf" || rc=1
   while IFS= read -r -d '' rec; do
     case "${rec:0:1}" in
@@ -264,9 +285,12 @@ count_lfs() {
   gd=$(g "$1" rev-parse --path-format=absolute --git-common-dir) || return 1
   st=$(g "$1" config --get lfs.storage) || st="lfs"
   case "$st" in /*) ;; ~*) st="$HOME${st#\~}" ;; *) st="$gd/$st" ;; esac
-  [ -e "$st/objects" ] || return 0
-  local tf frc; tf=$(mktemp)
-  find "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
+  # no store only on a definite ENOENT; EACCES on $st (or above) or a dangling objects symlink -> ERR
+  absent "$st/objects" && return 0
+  [ -e "$st/objects" ] || return 1
+  local tf frc; tf=$(tmpf)
+  # -H: follow objects itself if it's a symlink (a plain find would list the link, not the store)
+  find -H "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
   n=$(wc -c <"$tf"); rm -f "$tf"
   [ "$frc" = 0 ] || return 1
   lfs=$((lfs+n))
@@ -370,7 +394,7 @@ scan_entry() {   # $1 root as written, $2 display name, $3 abs path
   if [ -z "$(ls -A "$d" 2>/dev/null)" ]; then emit "$root" "$n" - - "$sz" "EMPTY$badname"; return; fi
   # git repos nested inside a non-git dir (kirby/katam): list them and audit each one
   # (find runs into a temp file so its exit status is visible: an unreadable subdir means "can't tell")
-  local nf ferr=""; nf=$(mktemp)
+  local nf ferr=""; nf=$(tmpf)
   find "$d" -mindepth 2 -maxdepth 6 -name .git -print0 -prune >"$nf" 2>/dev/null || ferr=" ERR"
   while IFS= read -r -d '' x; do nested+=("$(dirname "$x")"); done <"$nf"; rm -f "$nf"
   if [ ${#nested[@]} -gt 0 ]; then
@@ -384,10 +408,8 @@ scan_entry() {   # $1 root as written, $2 display name, $3 abs path
 for root in "$@"; do
   r="${root/#\~/$HOME}"; r="${r%/}"
   if [ ! -e "$r" ]; then
-    # only a definite "No such file" is a missing root; EACCES on a parent dir means we can't tell
-    # (message captured first: under pipefail, stat's own exit 1 would fail a `stat | grep` pipeline)
-    msg=$(LC_ALL=C stat -- "$r" 2>&1 >/dev/null)
-    if [[ "$msg" == *"No such file or directory"* ]] && [ ! -L "$r" ]; then
+    # only a definite ENOENT is a missing root; EACCES on a parent dir means we can't tell
+    if absent "$r"; then
       emit "$root" "(root)" - - - ROOT-MISSING
     else
       echo "#ROOTERR $root: can't tell whether it exists (permission denied on a parent, or a broken symlink)"
@@ -455,7 +477,7 @@ for id in "${ran[@]}"; do
     echo "== $id: audited ($n entries)"
     # warnings the scanner hit (the affected rows also carry ERR): never swallow them
     if [ -s "$tmp/$id.err" ]; then
-      echo "   !! scanner warnings on $id (first lines; the affected repos show ERR):"
+      echo "   !! warnings from the $id scan (git warnings also put ERR on the repo concerned):"
       head -5 "$tmp/$id.err" | sed 's/^/      /'
     fi
     # projects registered for this host but not found (only meaningful after a complete scan)
diff --git a/fleet/README.md b/fleet/README.md
index 8fdeb36..8c105ba 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -25,18 +25,19 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    - **Stop flags.** Any of these means the copy holds something not saved elsewhere: NOGIT, NESTED,
      NOREMOTE, LOCALREMOTE, SUBMODULE-NOBACKUP, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, HIDDENIDX,
      LFS, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run.
-     `IGNORED` and `LFS` stay on re-runs even when resolved, so for those two, a check you do yourself
-     (and note down) is what clears them:
+     `IGNORED` and `LFS` stay on re-runs, so they need a check you do yourself (and note down):
      - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
        Git can't vouch for it.
-     - `LFS:n`: the local LFS store stays after a successful upload, so the flag stays too. Run
-       `git lfs push --all <forge-remote>` (it uploads only what the server lacks). If that exits 0,
-       the LFS content is safe and this flag no longer blocks deleting *that* copy. Don't empty the
-       store just to make the flag go away.
+     - `LFS:n`: **keep the copy, or archive the whole repo directory first** (the tar recipe below,
+       including the `tar -df` check). A successful `git lfs push --all` is NOT enough: `lfs.url` can
+       send the upload somewhere other than the forge, and it skips LFS objects no commit points to
+       (e.g. a file staged and then reset). Both still exit 0. Don't empty the store to make the flag
+       go away.
      - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
        (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
-     - `REPLACE:n`: `git for-each-ref refs/replace` lists them. If a replacement holds content you
-       want, commit that content normally. Then `git replace -d <ref>` and re-run.
+     - `REPLACE:n`: `git replace -l` lists the replaced objects. If a replacement holds content you
+       want, commit that content normally. Then `git replace -d <sha>` (the replaced object's sha, as
+       `git replace -l` prints it) and re-run.
    - Deleting a whole root (e.g. `~/code`) also needs no `LOOSE:n` row for it.
 
    **What counts as "saved elsewhere":** only a remote on a forge (github.com, gitlab.com,
@@ -53,10 +54,15 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    - git-annex content.
 
    If any of that matters for a repo, archive the **whole repo directory** before deleting it,
-   `.git` included (that's where the reflog, LFS store and annex objects live), and check the archive:
+   `.git` included (that's where the reflog, LFS store and annex objects live), and check the archive
+   against the disk:
    ```bash
-   tar -C <parent> -czf <repo>-$(date +%F).tar.gz <repo> && tar -tzf <repo>-$(date +%F).tar.gz >/dev/null && echo OK
+   cd <parent-of-repo>                        # not inside the repo: tar would read its own output
+   a=~/archives/<repo>-$(date +%F).tar.gz     # outside every fleet root (or fleet-audit flags it LOOSE)
+   mkdir -p ~/archives && tar -czf "$a" <repo> && tar -dzf "$a" && echo VERIFIED
    ```
+   `tar -d` compares the archive with the files on disk. Only `VERIFIED` (exit 0, no differences)
+   counts. Then copy the archive somewhere that isn't this machine before deleting the repo.
    `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
    objects (LFS, annex).
 
@@ -145,17 +151,22 @@ some host was not, and 2 means a usage or inventory error.
 `UNPUSHED` counts commits reachable from any local ref (branches, tags, notes, replace refs, every
 worktree's HEAD; not the stash) that the forge remotes don't have. With `--fetch`, "have" means the
 server advertises it right now (`git ls-remote`). A branch that was squash-merged and then deleted on
-GitHub shows up here too. To see the actual commits before assuming work is at risk (forge remote
-`origin` here):
+GitHub shows up here too. To see the actual commits before assuming work is at risk, ask the
+server the same way the tool does (forge remote `origin` here). Don't use `--not --remotes=origin`,
+because cached tracking refs can be stale:
 ```bash
 git fetch --prune origin
-git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --not --remotes=origin
+git ls-remote origin | awk -F'\t' 'NF {print $1}' | git cat-file --batch-check='%(objectname) %(objecttype)' \
+  | awk '$2 ~ /^(commit|tag|tree|blob)$/ {print "^" $1}' \
+  | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin
 ```
 
 Keep `hosts.yaml` / `projects.yaml` in their flat format, because they are read with awk. Inline
-`# comments` are fine. In `hosts.yaml`, any line the parser doesn't understand (a tab, a host id at
-column 0, a stray top-level key, a missing status) stops the run with exit 2 before anything is
-scanned. In `projects.yaml` it's a warning.
+`# comments` are fine. In `hosts.yaml`, any line the parser doesn't understand stops the run with
+exit 2 before anything is scanned. That covers a tab in the indentation or content (trailing whitespace
+and comments are stripped first), a host id at column 0, a stray top-level key, a missing status, and a
+root that doesn't start with `/` or `~`. Roots are space-separated, so a path with spaces isn't
+supported. In `projects.yaml` it's a warning.
 
 ## Auth notes
 
```
