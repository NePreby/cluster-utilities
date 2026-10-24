# REVIEW_REQUEST: round 4, review of the fixes in `fleet` (commit 7594a87) in cluster-utilities

**Task: adversarial CODE REVIEW, scoped to what changed since round 3 (commit 980b7c9).** Round 3
found the problems listed below. The author says they're fixed. (1) Try to BREAK each fix with
concrete inputs. (2) Find bugs these changes introduced. Do not re-review unchanged code, except where a
change interacts with it. Do not propose new features. If both reviewers report no Blocking findings,
the branch will be merged into main.

## Context
cluster-utilities holds the owner's scripts for reaching compute, including a small fleet of machines
over Tailscale: a laptop and a lab workstation (ssh alias `lab`), with 3 more CachyOS lab workstations
about to be onboarded. All hosts run CachyOS. The owner's login shell is fish everywhere. There's no
YAML parser on the hosts (the YAML is read with awk). Password ssh login is intentionally LEFT ENABLED
(do not flag it). The real remotes are GitHub plus a few Overleaf (`https://git@git.overleaf.com/…`, which
need credentials, so `--fetch` reports FETCHFAIL for them; that's accepted). Several real repos use Git LFS.

- `bin/fleet-audit`: run from the laptop. Its output decides which copies are SAFE TO DELETE under the
  rule in `fleet/README.md` ("Before deleting a copy"). A false "clean" (unique work reported ok, or a
  host or entry silently not checked) is the worst bug class.
- `bin/fleet-onboard`: run on a NEW machine at its keyboard; it self-elevates with sudo. It must never
  cut the machine off the network or lock the owner out, and it must be idempotent.

Read in full: `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/README.md`. The diff since round 3 is
inlined at the end.

## Round-3 findings and the claimed fixes, to verify
- C1 hosts.yaml dropped hosts after a column-0 host id, a tab, or a stray top-level line (exit 0).
  **Claimed:** the file must be exactly one `hosts:` section. Tabs, other top-level lines, content
  before `hosts:` and a missing section are all exit 2.
- C2 An unreadable dir inside a repo: git status warns, exits 0, and the row looked clean. **Claimed:**
  any `git status` stderr means ERR. Scanner stderr is printed even for audited hosts.
- C3 An inaccessible parent made a root look ROOT-MISSING. **Claimed:** ROOT-MISSING only on an ENOENT
  message from `stat` (LC_ALL=C); otherwise `#ROOTERR`, which marks the host INCOMPLETE.
- C4 Stale tracking refs outside a narrowed refspec counted as pushed. **Claimed:** with `--fetch`, the
  negated set is the SHAs the server advertises (`git ls-remote`, filtered to locally present objects,
  fed to `rev-list --stdin` as `^sha`). An ls-remote failure gives FETCHFAIL.
- C5 Skip-worktree files whose names need quoting were missed. **Claimed:** `ls-files -v -z`, plus an
  explicit return status (which also fixes the spurious ERR on sparse checkouts).
- C6 LFS: a custom `lfs.storage` was ignored, and an unreadable store counted as 0. **Claimed:**
  `lfs.storage` is resolved; a failing `find` means ERR.
- C7 Blob replacement refs. **Claimed:** a `REPLACE:n` stop flag for any `refs/replace/*`.
- C8 The README called `git bundle --all` a full copy. **Claimed:** replaced with a tar of the whole repo
  dir plus a verification step.
- N3/N5 An empty forge host could match; push URLs weren't checked. **Claimed:** `is_forge` requires a
  non-empty host; all push URLs must be on a forge.
- N2/N4 README: how LFS/IGNORED are cleared; stale wording.
- fleet-onboard N6/N7: the auth key is now read after the upgrade + tun check; built-in TUN is accepted
  via /proc/config.gz.

## Also look for
- `rev-list --stdin` with `^sha` lines combined with `--exclude … --all` on the command line: correct
  semantics? Large advertisements (GitHub `refs/pull/*`), annotated tags (`^{}` lines), an empty
  advertisement (empty repo on the server), objects present but corrupt.
- Exit-status / pipefail / `set -e` traps in the new code, temp-file cleanup, `stat` message matching.
- Anything in the new README text that would make the deletion rule unsafe if followed literally.

## Deliverable
Write `reviews/7594a87/review_<model-slug>.md` containing:
- A table: round-3 item → FIXED / PARTIAL / BROKEN, with the input you tried.
- Findings in TWO separate sections, each ranked most severe first:
  - `## Blocking`: must be fixed before merge or use: data loss, owner lockout / machine cut off the
    network, or a false "clean" under the README rule. Say which of these it is.
  - `## Non-blocking`: safe to merge without it. Say in a line why it doesn't block.
  If a finding is borderline, put it under Blocking and say why. Each finding gives the file:line, what
  goes wrong, a CONCRETE trigger, and a suggested fix. Severity within a section: BLOCKER (Blocking
  only) · MAJOR · MINOR · NIT.
- What you checked and found correct.
- Bottom line: (a) fleet-onboard safe for the 3 machines? (b) fleet-audit trustworthy for deletion under
  the README rule? (c) ready to merge into main?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Diff since round 3 (980b7c9..7594a87, code + README)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index b0d322e..8d2346d 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -4,7 +4,7 @@
 # Scans each repo root (e.g. ~/Projects, ~/code) on every ACTIVE host in fleet/hosts.yaml, over
 # `ssh -o BatchMode=yes` (or locally for `ssh: local`), and checks each entry against
 # fleet/projects.yaml. Read-only (git runs with --no-optional-locks), except that --fetch runs
-# `git fetch --all --prune`. One row per entry that has something to act on:
+# `git fetch --prune <remote>` for each forge remote. One row per entry that has something to act on:
 #
 #   unsaved work (any of these means: do NOT delete this copy)
 #   NOGIT        not a git repo (only copy, unless you've backed it up)
@@ -12,7 +12,8 @@
 #   NOREMOTE     no remote at all          LOCALREMOTE  remotes, but none on a forge (github.com etc.):
 #                                                       a fleet box or local path isn't a backup
 #   UNPUSHED:n   commits reachable from any local ref (branches, tags, notes, replace, every worktree's
-#                HEAD) that no forge remote's tracking refs contain
+#                HEAD) that the forge remotes don't have. With --fetch that means "not among the
+#                SHAs the server advertises right now" (ls-remote); without it, cached tracking refs
 #   DETACHED     HEAD is detached (its commits are included in UNPUSHED)
 #   DIRTY:n      modified tracked files (incl. submodule changes)   UNTRACKED:n  untracked, not ignored
 #   STASH:n      stash entries
@@ -22,10 +23,12 @@
 #   WORKTREES:n  linked worktrees (their dirty/untracked are counted in; deleting this repo breaks them)
 #   SUBMODULES:n initialized submodules (their unpushed/dirty/untracked/stash are counted in)
 #   HIDDENIDX:n  files marked assume-unchanged / skip-worktree (on disk): status can't see their edits
-#   LFS:n        Git LFS file contents stored locally; the server may not have them all
+#   LFS:n        Git LFS file contents stored locally (honours lfs.storage); the server may not have
+#                them all. Re-running doesn't clear it: see fleet/README.md
+#   REPLACE:n    refs/replace/* entries; a replaced blob holds content no commit count sees
 #   SUBMODULE-NOBACKUP:path   a submodule with no forge remote
 #   FETCHFAIL    --fetch of a forge remote failed, so UNPUSHED/BEHIND come from stale refs
-#   ERR          a git command failed; the counts for this repo can't be trusted
+#   ERR          a git command failed or warned (e.g. a dir it couldn't open); counts can't be trusted
 #   LOOSE:n      files/symlinks sitting directly in a root (not in any repo dir)
 #   info
 #   BEHIND:n     behind upstream (current only with --fetch)    EMPTY  empty dir
@@ -37,10 +40,11 @@
 #
 #   fleet-audit                 # problems only, all active hosts
 #   fleet-audit --all           # every entry, including clean + correctly placed ones (as `ok`)
-#   fleet-audit --fetch         # git fetch --all --prune first, so UNPUSHED/BEHIND are current
+#   fleet-audit --fetch         # fetch + ask each forge what it has, so UNPUSHED/BEHIND are current
 #   fleet-audit --size          # add du sizes of each repo and of its ignored data (slow)
 #   fleet-audit lab             # only these host ids
 #   FLEET_AUDIT_TIMEOUT=3600 fleet-audit --fetch --size   # per-host time limit (default 600s, 1800s with --fetch)
+# (FLEET_AUDIT_FORGES / FLEET_AUDIT_GIT_SSH are test hooks and only apply to `ssh: local` hosts.)
 #
 # Exit: 0 every requested host fully audited · 1 some host NOT audited / incomplete · 2 usage/config error
 set -uo pipefail
@@ -75,9 +79,12 @@ hosts_all=$(awk -v US="$US" '
   function err(m) { print "hosts.yaml: " m > "/dev/stderr"; bad=1 }
   { line=$0; sub(/\r$/, "", line); sub(/[[:space:]]+#.*$/, "", line); sub(/^[[:space:]]*#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
   line == "" { next }
-  line == "hosts:" { inh=1; next }
-  line ~ /^[^ ]/ { flush(); inh=0; next }
-  !inh { next }
+  # The file is exactly one `hosts:` section. Anything else used to END the section silently, dropping
+  # every host after it (a host id at column 0, a tab-indented stanza, a stray key), so now it is an error.
+  line ~ /\t/ { err("line " NR ": tab character (indent with spaces): " $0); next }
+  line == "hosts:" { if (seenh) err("line " NR ": second hosts: section"); inh=1; seenh=1; next }
+  line ~ /^[^ ]/ { err("line " NR ": unexpected top-level line (only hosts: is allowed): " $0); next }
+  !inh { err("line " NR ": content before hosts:: " $0); next }
   line ~ /^  [A-Za-z0-9_.-]+:$/ {
     flush(); id=line; sub(/^  /, "", id); sub(/:$/, "", id); ssh=""; roots=""; st=""
     if (id in seen) err("host " id " defined twice"); seen[id]=1; next }
@@ -87,7 +94,7 @@ hosts_all=$(awk -v US="$US" '
     if (key == "ssh") ssh=val; else if (key == "roots") roots=val; else if (key == "status") st=tolower(val)
     next }
   { err("line " NR " not understood (expected 2-space host id or 4-space key: value): " $0) }
-  END { flush(); exit bad ? 2 : 0 }' "$HOSTS") || { echo "fix fleet/hosts.yaml first (nothing was scanned)" >&2; exit 2; }
+  END { flush(); if (!seenh) err("no hosts: section"); exit bad ? 2 : 0 }' "$HOSTS") || { echo "fix fleet/hosts.yaml first (nothing was scanned)" >&2; exit 2; }
 
 hosts_tsv=""; declare -A known=(); skipped=()
 while IFS="$US" read -r id ssh roots st; do
@@ -145,7 +152,10 @@ lines() { [ -z "$1" ] && echo 0 || printf '%s\n' "$1" | wc -l; }
 count_status() {
   local d=$1 tf rec p skip=0 rc=0
   tf=$(mktemp)
-  g "$d" status --porcelain -z --untracked-files=normal --ignored=matching --ignore-submodules=none >"$tf" || rc=1
+  # git only WARNS (exit 0) about dirs it can't open, e.g. a root-owned docker/sudo output dir, and the
+  # untracked files inside are then missing from the counts. So any stderr output means ERR.
+  g "$d" status --porcelain -z --untracked-files=normal --ignored=matching --ignore-submodules=none >"$tf" 2>"$tf.err" || rc=1
+  [ -s "$tf.err" ] && { rc=1; head -c 300 "$tf.err" | sed "s#^#$d: #" >&2; }
   while IFS= read -r -d '' rec; do
     if [ "$skip" = 1 ]; then skip=0; continue; fi
     case "$rec" in
@@ -155,7 +165,7 @@ count_status() {
       *) dirty=$((dirty+1)); [[ "${rec:0:2}" == *[RC]* ]] && skip=1 ;;
     esac
   done <"$tf"
-  rm -f "$tf"
+  rm -f "$tf" "$tf.err"
   return $rc
 }
 
@@ -171,15 +181,21 @@ url_host() {
   printf '%s' "$h" | tr '[:upper:]' '[:lower:]'
 }
 
-# remotes of repo $1 whose configured AND effective (insteadOf-rewritten) URL are on a forge
+is_forge() { local h; h=$(url_host "$1"); [ -n "$h" ] && [[ "$FORGES" == *" $h "* ]]; }
+
+# remotes of repo $1 whose configured URL, effective fetch URL (insteadOf) and every push URL
+# (pushurl / pushInsteadOf) are all on a forge
 backup_remotes() {
-  local d=$1 r u e rs
+  local d=$1 r u e pu rs ok
   rs=$(g "$d" remote) || return 1
   while IFS= read -r r; do
     [ -z "$r" ] && continue
     u=$(g "$d" config --get "remote.$r.url") || continue
     e=$(g "$d" remote get-url "$r") || return 1
-    if [[ "$FORGES" == *" $(url_host "$u") "* && "$FORGES" == *" $(url_host "$e") "* ]]; then echo "$r"; fi
+    pu=$(g "$d" remote get-url --push --all "$r") || return 1
+    ok=1; is_forge "$u" || ok=0; is_forge "$e" || ok=0
+    while IFS= read -r u; do [ -z "$u" ] || is_forge "$u" || ok=0; done <<<"$pu"
+    if [ "$ok" = 1 ]; then echo "$r"; fi
   done <<<"$rs"
   return 0
 }
@@ -195,34 +211,73 @@ fetch_backups() {
 }
 
 # commits reachable from ANY local ref (branches, tags, notes, replace, refs/original, every
-# worktree's HEAD; not stash, counted separately) that no backup remote's tracking refs contain.
-# Refs under refs/remotes/ of remotes that aren't backups (or no longer exist) prove nothing.
+# worktree's HEAD; not stash, counted separately) that the backup remotes don't have.
+#  - with --fetch: "have" = the SHAs each server ADVERTISES right now (git ls-remote). Local tracking
+#    refs can be stale in ways fetch never cleans up (a narrowed fetch refspec leaves origin/old
+#    behind; a changed URL keeps the old server's refs), so they aren't evidence.
+#  - without --fetch: the backup remotes' cached tracking refs (fine for a report, not for deleting).
+# Refs of remotes that aren't backups (or no longer exist) never count. Returns 1 on a git error,
+# 2 if a server couldn't be asked (-> FETCHFAIL).
 count_unpushed() {
-  local d=$1 nots=() r n
-  for r in $2; do nots+=("--remotes=$r"); done
-  n=$(g "$d" rev-list --count --exclude='refs/remotes/*' --exclude=refs/stash --all --not "${nots[@]}") || return 1
+  local d=$1 r n adv tf rc=0
+  tf=$(mktemp)
+  if [ "$fetch" = 1 ]; then
+    for r in $2; do
+      adv=$(timeout 60 git --no-optional-locks -C "$d" ls-remote "$r" </dev/null 2>/dev/null) || { rc=2; continue; }
+      # keep only objects we have locally (rev-list can't negate what it can't read); ^ = "not"
+      cut -f1 <<<"$adv" | sort -u | g "$d" cat-file --batch-check='%(objectname) %(objecttype)' 2>/dev/null \
+        | awk '$2 != "missing" {print "^" $1}' >>"$tf"
+    done
+    n=$(g "$d" rev-list --count --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin <"$tf") || { rm -f "$tf"; return 1; }
+  else
+    local nots=()
+    for r in $2; do nots+=("--remotes=$r"); done
+    n=$(g "$d" rev-list --count --exclude='refs/remotes/*' --exclude=refs/stash --all --not "${nots[@]}") || { rm -f "$tf"; return 1; }
+  fi
+  rm -f "$tf"
   unp=$((unp+n))
+  return $rc
 }
 
 # index entries whose edits git status won't show: assume-unchanged (lowercase tag) on any file,
 # skip-worktree (S) on a file that is actually on disk (sparse-checkout leaves those absent)
+# -z: names are never quoted, so a file with a tab or quote in its name is tested under its real name
 count_hidden() {
-  local d=$1 out line
-  out=$(g "$d" -c core.quotePath=false ls-files -v) || return 1
-  while IFS= read -r line; do
-    case "$line" in
-      [a-z]\ *) hid=$((hid+1)) ;;
-      S\ *) [ -e "$d/${line#S }" ] && hid=$((hid+1)) ;;
+  local d=$1 tf rec rc=0
+  tf=$(mktemp)
+  g "$d" ls-files -v -z >"$tf" || rc=1
+  while IFS= read -r -d '' rec; do
+    case "${rec:0:1}" in
+      [a-z]) hid=$((hid+1)) ;;
+      S) if [ -e "$d/${rec:2}" ] || [ -L "$d/${rec:2}" ]; then hid=$((hid+1)); fi ;;
     esac
-  done <<<"$out"
+  done <"$tf"
+  rm -f "$tf"
+  return $rc
 }
 
-# LFS file contents stored locally (the server may not have them if a push skipped the LFS hook)
+# LFS file contents stored locally (the server may not have them if a push skipped the LFS hook).
+# Storage is <git dir>/lfs unless lfs.storage moves it (relative values are inside the git dir).
+# A store that exists but can't be fully listed is an error, never "0 objects".
 count_lfs() {
-  local gd
+  local gd st n
   gd=$(g "$1" rev-parse --path-format=absolute --git-common-dir) || return 1
-  [ -d "$gd/lfs/objects" ] && lfs=$((lfs+$(find "$gd/lfs/objects" -type f | wc -l)))
-  return 0
+  st=$(g "$1" config --get lfs.storage) || st="lfs"
+  case "$st" in /*) ;; ~*) st="$HOME${st#\~}" ;; *) st="$gd/$st" ;; esac
+  [ -e "$st/objects" ] || return 0
+  local tf frc; tf=$(mktemp)
+  find "$st/objects" -type f -printf . >"$tf" 2>/dev/null; frc=$?   # one byte per object
+  n=$(wc -c <"$tf"); rm -f "$tf"
+  [ "$frc" = 0 ] || return 1
+  lfs=$((lfs+n))
+}
+
+# refs/replace/*: a replaced commit is caught by UNPUSHED, but a replaced BLOB (or tree) holds content
+# no commit count can see, so any replace ref is a stop flag
+count_replace() {
+  local n
+  n=$(g "$1" for-each-ref --format=x refs/replace/ | wc -l) || return 1
+  repl=$((repl+n))
 }
 
 # fields are \x1f-separated and never empty (bash `read` would shift them otherwise)
@@ -230,7 +285,7 @@ emit() { printf '%s\x1f%s\x1f%s\x1f%s\x1f%s\x1f%s\n' "${1:--}" "${2:--}" "${3:--
 
 scan_repo() {
   local root=$1 n=$2 repo=$3
-  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 hid=0 lfs=0 fetchfail=0
+  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 hid=0 lfs=0 repl=0 fetchfail=0 rc
   local ignpaths=() flags=() remotes backups r u wt line s p
   remotes=$(g "$repo" remote) || err=1
   backups=$(backup_remotes "$repo") || err=1
@@ -246,7 +301,11 @@ scan_repo() {
   count_status "$repo" || err=1
   count_hidden "$repo" || err=1
   count_lfs "$repo" || err=1
-  [ -n "$backups" ] && { count_unpushed "$repo" "$backups" || err=1; }
+  count_replace "$repo" || err=1
+  if [ -n "$backups" ]; then
+    count_unpushed "$repo" "$backups"; rc=$?
+    [ "$rc" = 1 ] && err=1; [ "$rc" = 2 ] && fetchfail=1
+  fi
   if g "$repo" rev-parse -q --verify '@{u}' >/dev/null 2>&1; then
     beh=$(g "$repo" rev-list --count 'HEAD..@{u}') || { beh=0; err=1; }
   fi
@@ -269,11 +328,13 @@ scan_repo() {
     count_status "$p" || err=1
     count_hidden "$p" || err=1
     count_lfs "$p" || err=1
+    count_replace "$p" || err=1
     u=$(backup_remotes "$p") || err=1
     if [ -z "$u" ]; then flags+=("SUBMODULE-NOBACKUP:${p#"$repo"/}")
     else
       if [ "$fetch" = 1 ]; then fetch_backups "$p" "$u" || fetchfail=1; fi
-      count_unpushed "$p" "$u" || err=1
+      count_unpushed "$p" "$u"; rc=$?
+      [ "$rc" = 1 ] && err=1; [ "$rc" = 2 ] && fetchfail=1
     fi
     u=$(g "$p" stash list) || err=1; st=$((st+$(lines "$u")))
   done <<<"$s"
@@ -288,6 +349,7 @@ scan_repo() {
   [ "$nsub" -gt 0 ] && flags+=("SUBMODULES:$nsub")
   [ "$hid" -gt 0 ] && flags+=("HIDDENIDX:$hid")
   [ "$lfs" -gt 0 ] && flags+=("LFS:$lfs")
+  [ "$repl" -gt 0 ] && flags+=("REPLACE:$repl")
   [ "$fetchfail" = 1 ] && flags+=(FETCHFAIL)
   [ "$err" = 1 ] && flags+=(ERR)
   [ "$beh" -gt 0 ] && flags+=("BEHIND:$beh")
@@ -321,7 +383,17 @@ scan_entry() {   # $1 root as written, $2 display name, $3 abs path
 
 for root in "$@"; do
   r="${root/#\~/$HOME}"; r="${r%/}"
-  if [ ! -e "$r" ]; then emit "$root" "(root)" - - - ROOT-MISSING; continue; fi
+  if [ ! -e "$r" ]; then
+    # only a definite "No such file" is a missing root; EACCES on a parent dir means we can't tell
+    # (message captured first: under pipefail, stat's own exit 1 would fail a `stat | grep` pipeline)
+    msg=$(LC_ALL=C stat -- "$r" 2>&1 >/dev/null)
+    if [[ "$msg" == *"No such file or directory"* ]] && [ ! -L "$r" ]; then
+      emit "$root" "(root)" - - - ROOT-MISSING
+    else
+      echo "#ROOTERR $root: can't tell whether it exists (permission denied on a parent, or a broken symlink)"
+    fi
+    continue
+  fi
   # exists but can't be listed: with nullglob that would look like an empty root, so the host's
   # audit is marked incomplete instead
   if [ ! -d "$r" ] || [ ! -r "$r" ] || [ ! -x "$r" ]; then echo "#ROOTERR $root: not a readable directory"; continue; fi
@@ -381,6 +453,11 @@ for id in "${ran[@]}"; do
   rooterr=$(sed -n 's/^#ROOTERR //p' "$tmp/$id.tsv")
   if [ "$rc" = 0 ] && [ "$(tail -1 "$tmp/$id.tsv")" = '#DONE' ] && [ -z "$rooterr" ]; then
     echo "== $id: audited ($n entries)"
+    # warnings the scanner hit (the affected rows also carry ERR): never swallow them
+    if [ -s "$tmp/$id.err" ]; then
+      echo "   !! scanner warnings on $id (first lines; the affected repos show ERR):"
+      head -5 "$tmp/$id.err" | sed 's/^/      /'
+    fi
     # projects registered for this host but not found (only meaningful after a complete scan)
     while IFS="$US" read -r p want; do
       case "$want" in '?'|'') continue ;; '*') ;; *) [[ ",$want," == *",$id,"* ]] || continue ;; esac
diff --git a/bin/fleet-onboard b/bin/fleet-onboard
index ad003bf..e16b22d 100755
--- a/bin/fleet-onboard
+++ b/bin/fleet-onboard
@@ -77,15 +77,8 @@ dig_ok() { dig +time=2 +tries=2 "$@" archlinux.org 2>/dev/null | grep -q 'status
 dns_ok() { getent hosts archlinux.org >/dev/null 2>&1 && dig_ok; }
 wait_dns() { local i; for i in $(seq 1 15); do dns_ok && return 0; sleep 1; done; return 1; }
 
-# Ask for the auth key up front (hidden, never in argv/history), so the rest runs unattended.
 keyfile=""
 trap '[ -z "$keyfile" ] || rm -f "$keyfile"' EXIT
-if [ "$authkey" = 1 ]; then
-  keyfile="$(mktemp)"; chmod 600 "$keyfile"
-  read -rsp "Tailscale auth key (input hidden): " key </dev/tty; echo
-  case "$key" in tskey-*) ;; *) echo "that doesn't look like a tskey-… auth key" >&2; exit 1 ;; esac
-  printf '%s' "$key" > "$keyfile"; unset key
-fi
 
 # 1. hostname
 if [ -n "$newname" ] && [ "$(hostnamectl --static)" != "$newname" ]; then
@@ -103,13 +96,25 @@ pacman -Syu --needed --noconfirm tailscale openssh git rsync tmux fish bind
 step "enable sshd + tailscaled"
 systemctl enable --now sshd
 systemctl enable tailscaled
-if [ ! -d /sys/module/tun ]; then
+# loaded module, or TUN built into the kernel (no /sys/module entry then)
+tun_ok() { [ -d /sys/module/tun ] || zgrep -qx 'CONFIG_TUN=y' /proc/config.gz 2>/dev/null; }
+if ! tun_ok; then
   echo "!! The kernel was upgraded and the tun module can't be loaded until you reboot." >&2
   echo "   Reboot, then rerun fleet-onboard (sshd and tailscaled will start at boot)." >&2
   exit 1
 fi
 systemctl start tailscaled
 
+# Ask for the auth key now (hidden, never in argv/history): after the upgrade and the tun check, so a
+# "reboot, then rerun" stop doesn't throw away a key you just typed. The rest runs unattended.
+if [ "$authkey" = 1 ]; then
+  keyfile="$(mktemp)"; chmod 600 "$keyfile"
+  read -rsp "Tailscale auth key (input hidden): " key </dev/tty; echo
+  case "$key" in tskey-*) ;; *) echo "that doesn't look like a tskey-… auth key" >&2; exit 1 ;; esac
+  printf '%s' "$key" > "$keyfile"; unset key
+fi
+
+
 # 3. SSH login menu. Source of truth: linux-config home/dot_config/fish/conf.d/ssh-session-menu.fish
 # (private repo, so a fresh box can't fetch it). Embedded copy; if you change the menu there, re-paste it here.
 read -r -d '' MENU <<'MENU_EOF' || true
diff --git a/fleet/README.md b/fleet/README.md
index 82a28f1..8fdeb36 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -24,18 +24,24 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
      UNLISTED, LEGACY, EMPTY (an empty dir: `rmdir` it). `UNDECIDED` means decide the owner first.
    - **Stop flags.** Any of these means the copy holds something not saved elsewhere: NOGIT, NESTED,
      NOREMOTE, LOCALREMOTE, SUBMODULE-NOBACKUP, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, HIDDENIDX,
-     LFS, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run:
+     LFS, REPLACE, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run.
+     `IGNORED` and `LFS` stay on re-runs even when resolved, so for those two, a check you do yourself
+     (and note down) is what clears them:
      - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
        Git can't vouch for it.
-     - `LFS:n`: run `git lfs push --all <forge-remote>` (it uploads only what the server lacks) and
-       check that it exits 0, or keep the copy.
+     - `LFS:n`: the local LFS store stays after a successful upload, so the flag stays too. Run
+       `git lfs push --all <forge-remote>` (it uploads only what the server lacks). If that exits 0,
+       the LFS content is safe and this flag no longer blocks deleting *that* copy. Don't empty the
+       store just to make the flag go away.
      - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
        (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
+     - `REPLACE:n`: `git for-each-ref refs/replace` lists them. If a replacement holds content you
+       want, commit that content normally. Then `git replace -d <ref>` and re-run.
    - Deleting a whole root (e.g. `~/code`) also needs no `LOOSE:n` row for it.
 
    **What counts as "saved elsewhere":** only a remote on a forge (github.com, gitlab.com,
-   git.overleaf.com, bitbucket.org, codeberg.org), checked by both its configured and its rewritten
-   URL. A copy on another fleet box (`user@lab:…`, `ssh://lab/…`) or a local path is not a backup.
+   git.overleaf.com, bitbucket.org, codeberg.org), checked by its configured URL, its rewritten
+   (insteadOf) URL and every push URL. A copy on another fleet box (`user@lab:…`, `ssh://lab/…`) or a local path is not a backup.
 
    **Not counted as rebuildable** (so they show up as IGNORED): `.ipynb_checkpoints`, `.cache`,
    `env/`. Only `.venv`, `venv`, `__pycache__`, `node_modules`, pytest/mypy/ruff caches, `.tox`/`.nox`
@@ -46,7 +52,13 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
    - annotated tag objects you never pushed, when they point at a commit that *is* pushed;
    - git-annex content.
 
-   If any of that matters for a repo, keep a full copy (`tar` or `git bundle create x.bundle --all`).
+   If any of that matters for a repo, archive the **whole repo directory** before deleting it,
+   `.git` included (that's where the reflog, LFS store and annex objects live), and check the archive:
+   ```bash
+   tar -C <parent> -czf <repo>-$(date +%F).tar.gz <repo> && tar -tzf <repo>-$(date +%F).tar.gz >/dev/null && echo OK
+   ```
+   `git bundle --all` is **not** a full copy: it leaves out reflog-only commits and anything outside git
+   objects (LFS, annex).
 
 ## Moving gitignored data to the owner host
 
@@ -130,12 +142,20 @@ FLEET_AUDIT_TIMEOUT=3600 fleet-audit --fetch --size   # raise the per-host time
 `fleet-audit --help` explains every flag. Exit 0 means every requested host was fully audited, 1 means
 some host was not, and 2 means a usage or inventory error.
 
-`UNPUSHED` counts commits reachable from local branches, tags or HEAD that no remote-tracking ref
-contains. A branch that was squash-merged and then deleted on GitHub shows up here too, so look at
-`git log --branches --tags HEAD --not --remotes` before assuming work is at risk.
+`UNPUSHED` counts commits reachable from any local ref (branches, tags, notes, replace refs, every
+worktree's HEAD; not the stash) that the forge remotes don't have. With `--fetch`, "have" means the
+server advertises it right now (`git ls-remote`). A branch that was squash-merged and then deleted on
+GitHub shows up here too. To see the actual commits before assuming work is at risk (forge remote
+`origin` here):
+```bash
+git fetch --prune origin
+git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --not --remotes=origin
+```
 
 Keep `hosts.yaml` / `projects.yaml` in their flat format, because they are read with awk. Inline
-`# comments` are fine. Lines the parser doesn't understand are reported on stderr, not silently dropped.
+`# comments` are fine. In `hosts.yaml`, any line the parser doesn't understand (a tab, a host id at
+column 0, a stray top-level key, a missing status) stops the run with exit 2 before anything is
+scanned. In `projects.yaml` it's a warning.
 
 ## Auth notes
 
```
