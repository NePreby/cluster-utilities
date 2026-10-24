# REVIEW_REQUEST: round 3, review of the fixes in `fleet` (commit 980b7c9) in cluster-utilities

**Task: adversarial CODE REVIEW, scoped to what changed since round 2 (commit 7a25a69).** Round 2
found the problems listed below. The author says they're fixed. (1) Try to BREAK each fix with
concrete inputs. (2) Find bugs these changes introduced. Do not re-review unchanged code, except where a
change interacts with it. Do not propose new features.

## Context
cluster-utilities holds the owner's scripts for reaching compute, including a small fleet of machines
over Tailscale: a laptop and a lab workstation (ssh alias `lab`), with 3 more CachyOS lab workstations
about to be onboarded. All hosts run CachyOS. The owner's login shell is fish everywhere. There's no
YAML parser on the hosts, so the YAML files are read with awk. Password ssh login is intentionally LEFT
ENABLED (do not flag it). The owner's real remotes are GitHub plus a few Overleaf (`https://git@git.overleaf.com/…`).
Several real repos use Git LFS.

- `bin/fleet-audit`: run from the laptop. Its output decides which copies are SAFE TO DELETE, under the
  rule in `fleet/README.md` ("Before deleting a copy"). A false "clean" (unique work reported ok, or a
  host or entry silently not checked) is the worst bug class.
- `bin/fleet-onboard`: run on a NEW machine at its keyboard; it self-elevates with sudo. It must never
  cut the machine off the network or lock the owner out, and it must be idempotent.

Read in full: `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/README.md`, `fleet/hosts.yaml`,
`fleet/projects.yaml`. The diff since round 2 is inlined at the end.

## Round-2 findings and the claimed fixes, to verify

fleet-audit, completeness:
- A1 A malformed active host was silently skipped with exit 0 (tab-IFS collapse). **Claimed:** hosts.yaml is
  parsed strictly in awk (unknown lines, missing or invalid status, an active host without ssh/roots, or a
  duplicate id mean exit 2, nothing scanned); records use `\x1f`; skipped hosts are listed on stdout.
- A2 The row parser had the same collapse. **Claimed:** `\x1f` and `emit` writes `-` for empty fields.
- A3 An unreadable root showed "audited (0 entries)"; an unreadable dir showed EMPTY. **Claimed:** `#ROOTERR`
  marks the host INCOMPLETE (exit 1); NOACCESS rows; a failing nested-repo `find` adds ERR.

fleet-audit, backup evidence:
- A4 LOCALREMOTE missed `user@host:path` and `ssh://host/…`. **Claimed:** only remotes whose configured AND
  insteadOf-effective URL host is on a forge allow-list count as a backup (`url_host`, `backup_remotes`).
- A5 Orphan `refs/remotes/<gone>/*` and `skipFetchAll` counted as pushed. **Claimed:** compare only
  against the backup remotes' refs (`--remotes=<name>`); `--fetch` fetches each backup remote by name.
- A6 Notes and replace refs were invisible. **Claimed:** `rev-list --exclude='refs/remotes/*'
  --exclude=refs/stash --all --not --remotes=<backup>…` with `GIT_NO_REPLACE_OBJECTS=1`.
- A7 `assume-unchanged`/`skip-worktree` edits were invisible. **Claimed:** a `HIDDENIDX:n` flag.
- A8 Local LFS payloads were invisible. **Claimed:** an `LFS:n` flag (objects in the common git dir).
- A9 REGEN hid `.ipynb_checkpoints`, `.cache` and `env/`. **Claimed:** removed from REGEN; README lists it.
- Also: `ignored-size` now parses `status -z`, with rename/copy source records skipped.

fleet-onboard:
- O1 The SSH refusal ran after `exec sudo` (env_reset drops SSH_CONNECTION). **Claimed:** `under_ssh` runs
  before the re-exec and again after it, walking the process tree for sshd/sshd-session/tailscaled.
- O-r1 A failing `tailscale status` aborted under set -e. **Claimed:** `|| true` plus a retry loop; no answer
  means a clear exit 1.
- O-r2 After a kernel upgrade, tun can't load. **Claimed:** `modprobe tun` before `-Syu`; if
  `/sys/module/tun` is missing afterwards, "reboot, then rerun".
- O-r3 getent never reads resolv.conf. **Claimed:** checks use `dig` (package `bind` added) through
  /etc/resolv.conf, and the switch requires `dig @127.0.0.53` to answer.
- O-r4 NeedsMachineAuth died before the hand-off. **Claimed:** an IP placeholder.
- O5 A failed resolved restart was masked. O6 A symlinked parent dir of the menu. O7 A relative
  resolv.conf link. **Claimed:** fixed.

## Also look for
- New false-clean paths created by the forge allow-list or by the `--all` minus backup-refs counting,
  e.g. a repo whose only forge remote has never been fetched (no tracking refs), a fork setup with
  `upstream` and `origin` on GitHub, Overleaf URLs, a remote named with a glob character, or a submodule
  checked out at a commit only on a non-forge remote.
- Exit-status and `set -e`/pipefail traps in the new code, `IFS="$US" read` edge cases, temp-file
  cleanup, `dig` behaviour on the real CachyOS defaults, `/proc/<pid>/stat` parsing, `realpath -m`.
- Anything that makes the README deletion rule unsafe even when the tool behaves as written.

## Deliverable
Write `reviews/980b7c9/review_<model-slug>.md` containing:
- A table: round-2 item → FIXED / PARTIAL / BROKEN, with the input you tried.
- Findings in TWO separate sections, each ranked most severe first:
  - `## Blocking`: must be fixed before this branch is merged or used, because it can cause data loss,
    lock the owner out or cut a machine off the network, or make fleet-audit report a false "clean"
    under the README rule. Say which of these it is.
  - `## Non-blocking`: safe to merge and use without it. Say in a line why it doesn't block.
  If a finding is borderline, put it under Blocking and say why. Each finding gives the file:line, what
  goes wrong, a CONCRETE trigger, and a suggested fix. Severity within a section: BLOCKER (Blocking
  only) · MAJOR · MINOR · NIT.
- What you checked and found correct.
- Bottom line: (a) is fleet-onboard safe to run on the 3 machines? (b) can fleet-audit's output be trusted
  for deletion decisions under the README rule? (c) is the branch ready to merge into main?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.

## Diff since round 2 (7a25a69..980b7c9, code + README)
```diff
diff --git a/bin/fleet-audit b/bin/fleet-audit
index e8f366b..b0d322e 100755
--- a/bin/fleet-audit
+++ b/bin/fleet-audit
@@ -9,17 +9,22 @@
 #   unsaved work (any of these means: do NOT delete this copy)
 #   NOGIT        not a git repo (only copy, unless you've backed it up)
 #   NESTED:n     a non-git dir holding n git repos; each also gets its own row (e.g. kirby/katam)
-#   NOREMOTE     git repo with no remote (commits exist only here)
-#   UNPUSHED:n   commits reachable from local branches, tags or HEAD that no remote-tracking ref has
+#   NOREMOTE     no remote at all          LOCALREMOTE  remotes, but none on a forge (github.com etc.):
+#                                                       a fleet box or local path isn't a backup
+#   UNPUSHED:n   commits reachable from any local ref (branches, tags, notes, replace, every worktree's
+#                HEAD) that no forge remote's tracking refs contain
 #   DETACHED     HEAD is detached (its commits are included in UNPUSHED)
 #   DIRTY:n      modified tracked files (incl. submodule changes)   UNTRACKED:n  untracked, not ignored
 #   STASH:n      stash entries
-#   IGNORED:n    gitignored files/dirs, minus regenerables (.venv, __pycache__, node_modules, caches).
+#   IGNORED:n    gitignored files/dirs, minus regenerables (.venv, venv, __pycache__, node_modules,
+#                pytest/mypy/ruff caches, .tox/.nox, *.egg-info).
 #                Usually DATA: copy it to the owner host before deleting. --size adds their du.
 #   WORKTREES:n  linked worktrees (their dirty/untracked are counted in; deleting this repo breaks them)
 #   SUBMODULES:n initialized submodules (their unpushed/dirty/untracked/stash are counted in)
-#   LOCALREMOTE  a remote is a local path or another fleet host, not a real backup
-#   FETCHFAIL    --fetch failed, so UNPUSHED/BEHIND come from stale refs
+#   HIDDENIDX:n  files marked assume-unchanged / skip-worktree (on disk): status can't see their edits
+#   LFS:n        Git LFS file contents stored locally; the server may not have them all
+#   SUBMODULE-NOBACKUP:path   a submodule with no forge remote
+#   FETCHFAIL    --fetch of a forge remote failed, so UNPUSHED/BEHIND come from stale refs
 #   ERR          a git command failed; the counts for this repo can't be trusted
 #   LOOSE:n      files/symlinks sitting directly in a root (not in any repo dir)
 #   info
@@ -54,48 +59,65 @@ for a in "$@"; do
   esac
 done
 
-# --- parse hosts.yaml -> "id<TAB>ssh<TAB>roots<TAB>status" for EVERY host ---
-# Every line is normalized first (inline `# …` comments + trailing whitespace stripped), so an
-# annotated host line (`  lab2:   # new box`) still starts a new host instead of merging into the last.
-hosts_all=$(awk '
-  function flush() { if (id != "") print id "\t" ssh "\t" roots "\t" st; id="" }
-  { line=$0; sub(/[[:space:]]+#.*$/, "", line); sub(/^[[:space:]]*#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
+# --- parse hosts.yaml -> "id US ssh US roots US status" for EVERY host (US = \x1f) ---
+# Strict on purpose: the deletion rule trusts "every host audited", so an inventory line the parser
+# doesn't understand is a config error (exit 2), never a host that quietly drops out. Every line is
+# normalized first (CR, inline `# …` comments, trailing whitespace), so `  lab2:   # new box` works.
+# \x1f instead of tabs: bash `read` collapses adjacent tabs, which shifted fields when one was empty.
+US=$'\x1f'
+hosts_all=$(awk -v US="$US" '
+  function flush() {
+    if (id == "") return
+    if (st == "") { err("host " id ": missing status:") }
+    else if (st != "active" && st != "planned" && st != "retired") { err("host " id ": status must be active|planned|retired, got " st) }
+    if (st == "active" && (ssh == "" || roots == "")) { err("active host " id ": needs ssh: and roots:") }
+    print id US ssh US roots US st; id="" }
+  function err(m) { print "hosts.yaml: " m > "/dev/stderr"; bad=1 }
+  { line=$0; sub(/\r$/, "", line); sub(/[[:space:]]+#.*$/, "", line); sub(/^[[:space:]]*#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
   line == "" { next }
   line == "hosts:" { inh=1; next }
   line ~ /^[^ ]/ { flush(); inh=0; next }
   !inh { next }
-  line ~ /^  [A-Za-z0-9_.-]+:$/ { flush(); id=line; sub(/^  /, "", id); sub(/:$/, "", id); ssh=""; roots=""; st=""; next }
-  line ~ /^    [a-z_]+:/ {
+  line ~ /^  [A-Za-z0-9_.-]+:$/ {
+    flush(); id=line; sub(/^  /, "", id); sub(/:$/, "", id); ssh=""; roots=""; st=""
+    if (id in seen) err("host " id " defined twice"); seen[id]=1; next }
+  id != "" && line ~ /^    [a-z_]+:/ {
     key=line; sub(/^    /, "", key); sub(/:.*/, "", key)
     val=line; sub(/^    [a-z_]+:[[:space:]]*/, "", val)
     if (key == "ssh") ssh=val; else if (key == "roots") roots=val; else if (key == "status") st=tolower(val)
     next }
-  { print "hosts.yaml:" NR ": unrecognized line, ignored: " $0 > "/dev/stderr" }
-  END { flush() }' "$HOSTS")
+  { err("line " NR " not understood (expected 2-space host id or 4-space key: value): " $0) }
+  END { flush(); exit bad ? 2 : 0 }' "$HOSTS") || { echo "fix fleet/hosts.yaml first (nothing was scanned)" >&2; exit 2; }
 
-hosts_tsv=""; declare -A known=()
-while IFS=$'\t' read -r id ssh roots st; do
+hosts_tsv=""; declare -A known=(); skipped=()
+while IFS="$US" read -r id ssh roots st; do
   [ -z "$id" ] && continue
-  [ -n "${known[$id]:-}" ] && { echo "hosts.yaml: host '$id' defined twice" >&2; exit 2; }
   known[$id]=$st
-  if [ "$st" != active ]; then echo "(skipping host '$id': status ${st:-<none>})" >&2; continue; fi
-  if [ -z "$ssh" ] || [ -z "$roots" ]; then echo "hosts.yaml: active host '$id' needs ssh: and roots:" >&2; exit 2; fi
-  hosts_tsv+="$id"$'\t'"$ssh"$'\t'"$roots"$'\n'
+  if [ "$st" != active ]; then skipped+=("$id ($st)"); continue; fi
+  hosts_tsv+="$id$US$ssh$US$roots"$'\n'
 done <<<"$hosts_all"
+[ -z "$hosts_tsv" ] && { echo "hosts.yaml: no active hosts" >&2; exit 2; }
 for h in "${only[@]}"; do
   if [ -z "${known[$h]:-}" ]; then echo "unknown host id '$h' (known: ${!known[*]})" >&2; exit 2; fi
   if [ "${known[$h]}" != active ]; then echo "host '$h' is not active (status: ${known[$h]})" >&2; exit 2; fi
 done
 
 # --- parse projects.yaml -> "name<TAB>host,host" ([*] = every host, [?] = undecided) ---
-proj_tsv=$(awk '
-  { line=$0; sub(/[[:space:]]+#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
+proj_tsv=$(awk -v US="$US" '
+  { line=$0; sub(/\r$/, "", line); sub(/[[:space:]]+#.*$/, "", line); sub(/^[[:space:]]*#.*$/, "", line); sub(/[[:space:]]+$/, "", line) }
+  line == "" || line == "projects:" { next }
   line ~ /^  [^ #][^:]*:[[:space:]]*\[[^]]*\]$/ {
     name=line; sub(/^  /, "", name); sub(/:.*/, "", name); gsub(/^["\047]|["\047]$/, "", name)
     hs=line; sub(/^[^\[]*\[/, "", hs); sub(/\].*/, "", hs); gsub(/[[:space:]]/, "", hs)
     if (name in seen) { print "projects.yaml:" NR ": duplicate entry " name " (first one wins)" > "/dev/stderr"; next }
-    seen[name]=1; print name "\t" hs; next }
-  line ~ /^  [^ #]/ { print "projects.yaml:" NR ": unrecognized line, ignored: " $0 > "/dev/stderr" }' "$PROJECTS")
+    seen[name]=1; print name US hs; next }
+  { print "projects.yaml:" NR ": unrecognized line, ignored: " $0 > "/dev/stderr" }' "$PROJECTS")
+while IFS="$US" read -r p hs; do
+  IFS=, read -ra hl <<<"$hs"
+  for h in "${hl[@]}"; do
+    case "$h" in '*'|'?') ;; *) [ -n "${known[$h]:-}" ] || echo "projects.yaml: $p lists unknown host id '$h'" >&2 ;; esac
+  done
+done <<<"$proj_tsv"
 
 # --- the per-host scanner (runs under bash on each host; args: fetch size root...) ---
 # Prints one TSV row per entry: root, name, branch, url, size, flags. Last line is `#DONE`: a scan
@@ -104,62 +126,127 @@ read -r -d '' SCAN <<'EOF'
 fetch=$1; size=$2; shift 2
 set -o pipefail
 shopt -s nullglob dotglob
-export GIT_TERMINAL_PROMPT=0 GIT_SSH_COMMAND='ssh -o BatchMode=yes -o ConnectTimeout=15'
-REGEN='(^|/)(\.venv|venv|env|__pycache__|node_modules|\.pytest_cache|\.mypy_cache|\.ruff_cache|\.ipynb_checkpoints|\.tox|\.nox|\.cache|[^/]*\.egg-info|\.DS_Store)/?$'
+# GIT_NO_REPLACE_OBJECTS: walk the real history; replace refs themselves are still counted via --all
+export GIT_TERMINAL_PROMPT=0 GIT_NO_REPLACE_OBJECTS=1
+export GIT_SSH_COMMAND="${FLEET_AUDIT_GIT_SSH:-ssh -o BatchMode=yes -o ConnectTimeout=15}"
+# Ignored dirs assumed regenerable (not counted in IGNORED). Keep this list to things that can be
+# rebuilt from the repo; .ipynb_checkpoints, .cache and env/ are NOT here because they can hold the
+# only copy of something (fleet/README.md lists these too).
+REGEN='(^|/)(\.venv|venv|__pycache__|node_modules|\.pytest_cache|\.mypy_cache|\.ruff_cache|\.tox|\.nox|[^/]*\.egg-info|\.DS_Store)/?$'
+# Only these hosts count as an independent backup. A remote on a fleet box (user@lab:…, ssh://lab/…),
+# a local path or anything else is not, and gets LOCALREMOTE if it's all the repo has.
+FORGES=" ${FLEET_AUDIT_FORGES:-github.com gitlab.com git.overleaf.com bitbucket.org codeberg.org} "
 g() { local d=$1; shift; git --no-optional-locks -C "$d" "$@"; }
 lines() { [ -z "$1" ] && echo 0 || printf '%s\n' "$1" | wc -l; }
 
-# add one worktree's status into dirty/unt/ign (+ ignpaths); 1 on git error
+# add one worktree's status into dirty/unt/ign (+ ignpaths); 1 on git error.
+# -z (NUL-separated, never quoted) so paths with spaces or odd bytes stay exact for du; a rename or
+# copy entry (R/C in the XY code) is followed by one extra record, the source path, which is skipped.
 count_status() {
-  local d=$1 out line p
-  out=$(g "$d" status --porcelain --untracked-files=normal --ignored=matching --ignore-submodules=none) || return 1
-  while IFS= read -r line; do
-    [ -z "$line" ] && continue
-    case "$line" in
+  local d=$1 tf rec p skip=0 rc=0
+  tf=$(mktemp)
+  g "$d" status --porcelain -z --untracked-files=normal --ignored=matching --ignore-submodules=none >"$tf" || rc=1
+  while IFS= read -r -d '' rec; do
+    if [ "$skip" = 1 ]; then skip=0; continue; fi
+    case "$rec" in
       '?? '*) unt=$((unt+1)) ;;
-      '!! '*) p=${line#!! }; p=${p%/}
+      '!! '*) p=${rec#!! }; p=${p%/}
               if [[ ! "$p" =~ $REGEN ]]; then ign=$((ign+1)); ignpaths+=("$d/$p"); fi ;;
-      *) dirty=$((dirty+1)) ;;
+      *) dirty=$((dirty+1)); [[ "${rec:0:2}" == *[RC]* ]] && skip=1 ;;
     esac
-  done <<<"$out"
+  done <"$tf"
+  rm -f "$tf"
+  return $rc
+}
+
+# host part of a git URL (lowercased); empty for a local path
+url_host() {
+  local u=$1 h
+  case "$u" in
+    file://*|/*|./*|../*|~*) return ;;
+    *://*) h=${u#*://}; h=${h%%/*}; h=${h##*@}; h=${h%%:*} ;;
+    *:*) h=${u%%:*}; h=${h##*@} ;;          # scp-style [user@]host:path
+    *) return ;;
+  esac
+  printf '%s' "$h" | tr '[:upper:]' '[:lower:]'
+}
+
+# remotes of repo $1 whose configured AND effective (insteadOf-rewritten) URL are on a forge
+backup_remotes() {
+  local d=$1 r u e rs
+  rs=$(g "$d" remote) || return 1
+  while IFS= read -r r; do
+    [ -z "$r" ] && continue
+    u=$(g "$d" config --get "remote.$r.url") || continue
+    e=$(g "$d" remote get-url "$r") || return 1
+    if [[ "$FORGES" == *" $(url_host "$u") "* && "$FORGES" == *" $(url_host "$e") "* ]]; then echo "$r"; fi
+  done <<<"$rs"
+  return 0
+}
+
+# --fetch: fetch each backup remote by name (a `fetch --all` silently skips remotes with
+# skipFetchAll), pruning deleted branches. 1 if any fetch failed.
+fetch_backups() {
+  local d=$1 r rc=0
+  for r in $2; do
+    timeout 90 git --no-optional-locks -C "$d" fetch --prune -q "$r" </dev/null >/dev/null 2>&1 || rc=1
+  done
+  return $rc
 }
 
-# commits not on any remote-tracking ref: local branches + tags + HEAD
+# commits reachable from ANY local ref (branches, tags, notes, replace, refs/original, every
+# worktree's HEAD; not stash, counted separately) that no backup remote's tracking refs contain.
+# Refs under refs/remotes/ of remotes that aren't backups (or no longer exist) prove nothing.
 count_unpushed() {
-  local d=$1 heads=(--branches --tags) n
-  g "$d" rev-parse -q --verify HEAD >/dev/null 2>&1 && heads+=(HEAD)
-  n=$(g "$d" rev-list --count "${heads[@]}" --not --remotes) || return 1
+  local d=$1 nots=() r n
+  for r in $2; do nots+=("--remotes=$r"); done
+  n=$(g "$d" rev-list --count --exclude='refs/remotes/*' --exclude=refs/stash --all --not "${nots[@]}") || return 1
   unp=$((unp+n))
 }
 
-emit() { printf '%s\t%s\t%s\t%s\t%s\t%s\n' "$1" "$2" "$3" "$4" "$5" "$6"; }
+# index entries whose edits git status won't show: assume-unchanged (lowercase tag) on any file,
+# skip-worktree (S) on a file that is actually on disk (sparse-checkout leaves those absent)
+count_hidden() {
+  local d=$1 out line
+  out=$(g "$d" -c core.quotePath=false ls-files -v) || return 1
+  while IFS= read -r line; do
+    case "$line" in
+      [a-z]\ *) hid=$((hid+1)) ;;
+      S\ *) [ -e "$d/${line#S }" ] && hid=$((hid+1)) ;;
+    esac
+  done <<<"$out"
+}
+
+# LFS file contents stored locally (the server may not have them if a push skipped the LFS hook)
+count_lfs() {
+  local gd
+  gd=$(g "$1" rev-parse --path-format=absolute --git-common-dir) || return 1
+  [ -d "$gd/lfs/objects" ] && lfs=$((lfs+$(find "$gd/lfs/objects" -type f | wc -l)))
+  return 0
+}
+
+# fields are \x1f-separated and never empty (bash `read` would shift them otherwise)
+emit() { printf '%s\x1f%s\x1f%s\x1f%s\x1f%s\x1f%s\n' "${1:--}" "${2:--}" "${3:--}" "${4:--}" "${5:--}" "${6:--}"; }
 
 scan_repo() {
   local root=$1 n=$2 repo=$3
-  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 fetchfail=0 localrem=0
-  local ignpaths=() flags=() remotes r u wt line s
+  local url=- br sz=- err=0 dirty=0 unt=0 ign=0 unp=0 beh=0 st=0 wts=0 nsub=0 hid=0 lfs=0 fetchfail=0
+  local ignpaths=() flags=() remotes backups r u wt line s p
   remotes=$(g "$repo" remote) || err=1
+  backups=$(backup_remotes "$repo") || err=1
   if [ -n "$remotes" ]; then
-    url=$(g "$repo" remote get-url "$(head -1 <<<"$remotes")" | sed -E 's#^(https://github.com/|git@github.com:)##; s#\.git$##') || err=1
-    while IFS= read -r r; do
-      # the URL as configured (not insteadOf-rewritten): that's what says where the backup lives
-      u=$(g "$repo" config --get "remote.$r.url") || { err=1; continue; }
-      case "$u" in
-        *://*) case "$u" in file://*) localrem=1 ;; esac ;;   # https://, ssh://
-        *@*:*) ;;                                            # scp-style user@host:path (GitHub etc.)
-        *) localrem=1 ;;                                     # /path, ../path, or host:path (a fleet box)
-      esac
-    done <<<"$remotes"
-    if [ "$fetch" = 1 ]; then
-      timeout 90 git --no-optional-locks -C "$repo" fetch --all --prune -q </dev/null >/dev/null 2>&1 || fetchfail=1
-    fi
+    r=$(head -1 <<<"${backups:-$remotes}")
+    url=$(g "$repo" config --get "remote.$r.url" | sed -E 's#^(https://github.com/|git@github.com:)##; s#\.git$##') || err=1
   fi
+  if [ "$fetch" = 1 ] && [ -n "$backups" ]; then fetch_backups "$repo" "$backups" || fetchfail=1; fi
   if br=$(g "$repo" symbolic-ref -q --short HEAD); then :; else
     if [ $? -eq 1 ]; then br=DETACHED; flags+=(DETACHED); else br=?; err=1; fi
   fi
 
   count_status "$repo" || err=1
-  [ -n "$remotes" ] && { count_unpushed "$repo" || err=1; }
+  count_hidden "$repo" || err=1
+  count_lfs "$repo" || err=1
+  [ -n "$backups" ] && { count_unpushed "$repo" "$backups" || err=1; }
   if g "$repo" rev-parse -q --verify '@{u}' >/dev/null 2>&1; then
     beh=$(g "$repo" rev-list --count 'HEAD..@{u}') || { beh=0; err=1; }
   fi
@@ -170,7 +257,7 @@ scan_repo() {
   while IFS= read -r line; do
     case "$line" in worktree\ *) p=${line#worktree }
       [ "$p" = "$(realpath "$repo")" ] && continue
-      wts=$((wts+1)); [ -d "$p" ] && { count_status "$p" || err=1; } ;;
+      wts=$((wts+1)); [ -d "$p" ] && { count_status "$p" || err=1; count_hidden "$p" || err=1; } ;;
     esac
   done <<<"$wt"
 
@@ -180,11 +267,18 @@ scan_repo() {
     [ -z "$p" ] && continue
     nsub=$((nsub+1))
     count_status "$p" || err=1
-    count_unpushed "$p" || err=1
+    count_hidden "$p" || err=1
+    count_lfs "$p" || err=1
+    u=$(backup_remotes "$p") || err=1
+    if [ -z "$u" ]; then flags+=("SUBMODULE-NOBACKUP:${p#"$repo"/}")
+    else
+      if [ "$fetch" = 1 ]; then fetch_backups "$p" "$u" || fetchfail=1; fi
+      count_unpushed "$p" "$u" || err=1
+    fi
     u=$(g "$p" stash list) || err=1; st=$((st+$(lines "$u")))
   done <<<"$s"
 
-  [ -z "$remotes" ] && flags+=(NOREMOTE)
+  if [ -z "$remotes" ]; then flags+=(NOREMOTE); elif [ -z "$backups" ]; then flags+=(LOCALREMOTE); fi
   [ "$unp" -gt 0 ] && flags+=("UNPUSHED:$unp")
   [ "$dirty" -gt 0 ] && flags+=("DIRTY:$dirty")
   [ "$unt" -gt 0 ] && flags+=("UNTRACKED:$unt")
@@ -192,7 +286,8 @@ scan_repo() {
   [ "$ign" -gt 0 ] && flags+=("IGNORED:$ign")
   [ "$wts" -gt 0 ] && flags+=("WORKTREES:$wts")
   [ "$nsub" -gt 0 ] && flags+=("SUBMODULES:$nsub")
-  [ "$localrem" = 1 ] && flags+=(LOCALREMOTE)
+  [ "$hid" -gt 0 ] && flags+=("HIDDENIDX:$hid")
+  [ "$lfs" -gt 0 ] && flags+=("LFS:$lfs")
   [ "$fetchfail" = 1 ] && flags+=(FETCHFAIL)
   [ "$err" = 1 ] && flags+=(ERR)
   [ "$beh" -gt 0 ] && flags+=("BEHIND:$beh")
@@ -205,25 +300,31 @@ scan_repo() {
 
 scan_entry() {   # $1 root as written, $2 display name, $3 abs path
   local root=$1 n=$2 d=$3 top nested=() sz=- x
-  if [[ "$n" == *$'\t'* || "$n" == *$'\n'* ]]; then n=$(printf '%q' "$n"); badname=" BADNAME"; else badname=""; fi
+  if [[ "$n" == *$'\t'* || "$n" == *$'\n'* || "$n" == *$'\x1f'* ]]; then n=$(printf '%q' "$n"); badname=" BADNAME"; else badname=""; fi
+  if [ ! -r "$d" ] || [ ! -x "$d" ]; then emit "$root" "$n" - - - "ERR NOACCESS$badname"; return; fi
   top=$(git --no-optional-locks -C "$d" rev-parse --show-toplevel 2>/dev/null)
   if [ -n "$top" ] && [ "$top" = "$(realpath "$d")" ]; then scan_repo "$root" "$n" "$d"; return; fi
   [ "$size" = 1 ] && sz=$(du -sh "$d" 2>/dev/null | cut -f1)
   if [ -z "$(ls -A "$d" 2>/dev/null)" ]; then emit "$root" "$n" - - "$sz" "EMPTY$badname"; return; fi
   # git repos nested inside a non-git dir (kirby/katam): list them and audit each one
-  while IFS= read -r -d '' x; do nested+=("$(dirname "$x")"); done \
-    < <(find "$d" -mindepth 2 -maxdepth 4 -name .git -print0 -prune 2>/dev/null)
+  # (find runs into a temp file so its exit status is visible: an unreadable subdir means "can't tell")
+  local nf ferr=""; nf=$(mktemp)
+  find "$d" -mindepth 2 -maxdepth 6 -name .git -print0 -prune >"$nf" 2>/dev/null || ferr=" ERR"
+  while IFS= read -r -d '' x; do nested+=("$(dirname "$x")"); done <"$nf"; rm -f "$nf"
   if [ ${#nested[@]} -gt 0 ]; then
-    emit "$root" "$n" - - "$sz" "NOGIT NESTED:${#nested[@]}$badname"
+    emit "$root" "$n" - - "$sz" "NOGIT NESTED:${#nested[@]}$ferr$badname"
     for x in "${nested[@]}"; do scan_repo "$root" "$n/${x#"$d"/}" "$x"; done
   else
-    emit "$root" "$n" - - "$sz" "NOGIT$badname"
+    emit "$root" "$n" - - "$sz" "NOGIT$ferr$badname"
   fi
 }
 
 for root in "$@"; do
   r="${root/#\~/$HOME}"; r="${r%/}"
-  if [ ! -d "$r" ]; then emit "$root" "(root)" - - - ROOT-MISSING; continue; fi
+  if [ ! -e "$r" ]; then emit "$root" "(root)" - - - ROOT-MISSING; continue; fi
+  # exists but can't be listed: with nullglob that would look like an empty root, so the host's
+  # audit is marked incomplete instead
+  if [ ! -d "$r" ] || [ ! -r "$r" ] || [ ! -x "$r" ]; then echo "#ROOTERR $root: not a readable directory"; continue; fi
   loose=0
   for e in "$r"/*; do
     if [ -L "$e" ] || [ ! -d "$e" ]; then loose=$((loose+1)); continue; fi
@@ -237,7 +338,7 @@ EOF
 tmp=$(mktemp -d); trap 'rm -rf "$tmp"' EXIT
 tmo=600; [ "$fetch" = 1 ] && tmo=1800; tmo=${FLEET_AUDIT_TIMEOUT:-$tmo}   # per-host scan timeout (s)
 ran=()
-while IFS=$'\t' read -r id ssh roots; do
+while IFS="$US" read -r id ssh roots; do
   [ -z "$id" ] && continue
   if [ ${#only[@]} -gt 0 ] && [[ ! " ${only[*]} " =~ " $id " ]]; then continue; fi
   ran+=("$id")
@@ -259,10 +360,10 @@ wait
 failed=0
 printf '%-40s %-7s %-11s %-24s %-6s %s\n' DIR HOST ROOT BRANCH SIZE FLAGS
 for id in "${ran[@]}"; do
-  grep -v '^#DONE$' "$tmp/$id.tsv" | while IFS=$'\t' read -r root n br url sz fl; do
-    flags=($fl)
+  grep -v '^#' "$tmp/$id.tsv" | while IFS="$US" read -r root n br url sz fl; do
+    flags=($fl); [ "$fl" = - ] && flags=()
     top=${n%%/*}
-    want=$(awk -F'\t' -v n="$top" '$1==n {print $2}' <<<"$proj_tsv")
+    want=$(awk -F"$US" -v n="$top" '$1==n {print $2}' <<<"$proj_tsv")
     if [[ "$n" == "(root)" || "$n" == "(loose files)" ]]; then :
     elif [ -z "$want" ]; then flags+=(UNLISTED)
     elif [ "$want" = "?" ]; then flags+=(UNDECIDED)
@@ -276,23 +377,27 @@ done | sort -k1,1 -k2,2
 
 echo
 for id in "${ran[@]}"; do
-  rc=$(cat "$tmp/$id.rc" 2>/dev/null || echo '?'); n=$(grep -vc '^#DONE$' "$tmp/$id.tsv")
-  if [ "$rc" = 0 ] && [ "$(tail -1 "$tmp/$id.tsv")" = '#DONE' ]; then
+  rc=$(cat "$tmp/$id.rc" 2>/dev/null || echo '?'); n=$(grep -vc '^#' "$tmp/$id.tsv")
+  rooterr=$(sed -n 's/^#ROOTERR //p' "$tmp/$id.tsv")
+  if [ "$rc" = 0 ] && [ "$(tail -1 "$tmp/$id.tsv")" = '#DONE' ] && [ -z "$rooterr" ]; then
     echo "== $id: audited ($n entries)"
     # projects registered for this host but not found (only meaningful after a complete scan)
-    while IFS=$'\t' read -r p want; do
+    while IFS="$US" read -r p want; do
       case "$want" in '?'|'') continue ;; '*') ;; *) [[ ",$want," == *",$id,"* ]] || continue ;; esac
-      awk -F'\t' -v p="$p" '$2==p {f=1} END {exit !f}' "$tmp/$id.tsv" || echo "   -- $p: registered for $id but not found there"
+      awk -F"$US" -v p="$p" '$2==p {f=1} END {exit !f}' "$tmp/$id.tsv" || echo "   -- $p: registered for $id but not found there"
     done <<<"$proj_tsv"
   else
     failed=1
     why="exit $rc"; [ "$rc" = 124 ] && why="timed out after ${tmo}s"; [ "$rc" = 255 ] && why="ssh failed"
-    if [ "$n" -gt 0 ]; then
+    if [ -n "$rooterr" ]; then
+      echo "!! $id: INCOMPLETE, root(s) could not be listed: $(tr '\n' ';' <<<"$rooterr")"
+    elif [ "$n" -gt 0 ]; then
       echo "!! $id: INCOMPLETE ($why) after $n entries. Entries not shown above were NOT checked, not clean"
     else
       echo "!! $id: NOT AUDITED ($why): $(tr '\n' ' ' < "$tmp/$id.err" | head -c 200)"
     fi
   fi
 done
+for sk in "${skipped[@]}"; do echo "-- skipped host $sk: not audited"; done
 [ "$fetch" = 0 ] && echo "note: no --fetch, so UNPUSHED/BEHIND use cached remote refs. Run with --fetch before deleting anything."
 exit "$failed"
diff --git a/bin/fleet-onboard b/bin/fleet-onboard
index 2e19245..ad003bf 100755
--- a/bin/fleet-onboard
+++ b/bin/fleet-onboard
@@ -7,20 +7,21 @@
 #
 # Idempotent; rerunning skips whatever is already done. Self-elevates with sudo (one prompt). It:
 #   1. sets the hostname (that becomes the Tailscale device name), if you pass one
-#   2. full system upgrade + installs git, rsync, tmux, fish, openssh, tailscale (`pacman -Syu`: a bare
-#      -S against a stale sync DB 404s or half-upgrades openssh); enables sshd and tailscaled
+#   2. full system upgrade + installs git, rsync, tmux, fish, openssh, tailscale, bind (for `dig`)
+#      (`pacman -Syu`: a bare -S against a stale sync DB 404s or half-upgrades openssh); enables sshd
+#      and tailscaled. If the upgrade replaced the kernel so tun can't load, it says: reboot, rerun.
 #   3. installs the SSH login menu (plain fish / tmux / herdr, plain after 5s) into the user's
 #      ~/.config/fish/conf.d/, if their login shell is fish
 #   4. applies the CachyOS MagicDNS fix: /etc/resolv.conf -> systemd-resolved stub, so that
 #      NetworkManager stops overwriting it (https://tailscale.com/s/resolved-nm). Guarded: it only
-#      switches if DNS works before, the stub exists and the backup succeeded; afterwards it checks DNS
-#      and puts the old resolv.conf back (and exits 1) if resolution broke.
+#      switches if DNS works before, the stub listener answers and the backup succeeded; afterwards it
+#      checks DNS through /etc/resolv.conf itself (dig) and puts the old file back (exit 1) if it broke.
 #   5. runs `tailscale up` unless the node is already logged in. An auth key never reaches argv or
 #      shell history: it's read hidden from the tty into a 0600 temp file, passed as --auth-key=file:…
 #   6. prints what to paste on the laptop (ssh alias, ssh-copy-id, hosts.yaml entry)
 #
-# Over SSH it refuses without --force: steps 4–5 restart NetworkManager and tailscaled, which can cut the
-# session it's running in and leave it half-done.
+# Over SSH (SSH_CONNECTION, or an sshd/tailscaled parent process) it refuses without --force: steps
+# 4–5 restart NetworkManager and tailscaled, which can cut the session and leave it half-done.
 #
 # It does NOT touch sshd auth settings: password login stays on, and key login works once
 # ssh-copy-id has run. (For key-only there's bin/harden-sshd-keyonly, but that's optional.)
@@ -29,11 +30,6 @@ set -euo pipefail
 
 case " $* " in *" -h "*|*" --help "*) awk 'NR>1 && /^#/ {sub(/^# ?/,""); print; next} NR>1 {exit}' "$0"; exit 0 ;; esac
 
-if [ "$(id -u)" -ne 0 ]; then
-  # via bash + absolute path, so it still works when started as `bash fleet-onboard` / without +x
-  exec sudo -- bash "$(readlink -f "$0")" "$@"
-fi
-
 authkey=0; force=0; newname=""
 for a in "$@"; do
   case "$a" in
@@ -44,19 +40,41 @@ for a in "$@"; do
   esac
 done
 
+# Over SSH? Checked BEFORE the sudo re-exec (sudo's env_reset drops SSH_CONNECTION) and again after
+# it via the process tree, which sudo can't hide: any sshd / tailscaled (Tailscale SSH) ancestor.
+under_ssh() {
+  [ -n "${SSH_CONNECTION:-}" ] && return 0
+  local pid=$$ comm
+  while [ "${pid:-0}" -gt 1 ]; do
+    comm=$(cat "/proc/$pid/comm" 2>/dev/null) || return 1
+    case "$comm" in sshd|sshd-session|sshd-auth|tailscaled) return 0 ;; esac
+    pid=$(sed 's/.*) //' "/proc/$pid/stat" 2>/dev/null | cut -d' ' -f2) || return 1
+  done
+  return 1
+}
+if [ "$force" = 0 ] && under_ssh; then
+  echo "Refusing over SSH: the DNS and tailscale steps restart NetworkManager/tailscaled and can drop this" >&2
+  echo "session partway. Run it at the machine's keyboard, or pass --force if you accept that." >&2
+  exit 1
+fi
+
+if [ "$(id -u)" -ne 0 ]; then
+  # via bash + absolute path, so it still works when started as `bash fleet-onboard` / without +x
+  exec sudo -- bash "$(readlink -f "$0")" "$@"
+fi
+
 user="${SUDO_USER:-}"
 if [ -z "$user" ] || [ "$user" = root ]; then
   echo "Run this as your normal user (it sudo's itself), not from a root login: it needs to know whose" >&2
   echo "home gets the ssh menu and which user goes in the ssh alias." >&2
   exit 1
 fi
-if [ -n "${SSH_CONNECTION:-}" ] && [ "$force" = 0 ]; then
-  echo "Refusing over SSH: the DNS and tailscale steps restart NetworkManager/tailscaled and can drop this" >&2
-  echo "session partway. Run it at the machine's keyboard, or pass --force if you accept that." >&2
-  exit 1
-fi
 step() { printf '\n== %s\n' "$*"; }
-dns_ok() { getent hosts archlinux.org >/dev/null 2>&1 || getent hosts github.com >/dev/null 2>&1; }
+# DNS checks. getent goes through nss-resolve, which talks to systemd-resolved directly and never
+# reads /etc/resolv.conf, so on its own it can't tell whether the resolv.conf switch works. dig
+# (package `bind`) does read /etc/resolv.conf, and `dig @127.0.0.53` tests the stub listener itself.
+dig_ok() { dig +time=2 +tries=2 "$@" archlinux.org 2>/dev/null | grep -q 'status: NOERROR'; }
+dns_ok() { getent hosts archlinux.org >/dev/null 2>&1 && dig_ok; }
 wait_dns() { local i; for i in $(seq 1 15); do dns_ok && return 0; sleep 1; done; return 1; }
 
 # Ask for the auth key up front (hidden, never in argv/history), so the rest runs unattended.
@@ -77,10 +95,20 @@ fi
 host="$(hostnamectl --static)"
 
 # 2. packages + services
-step "pacman -Syu + tailscale openssh git rsync tmux fish"
-pacman -Syu --needed --noconfirm tailscale openssh git rsync tmux fish
+# tailscaled needs the tun module (CONFIG_TUN=m on linux-cachyos). Load it BEFORE the upgrade:
+# if -Syu installs a new kernel, the running kernel's modules are gone and tun can't load until reboot.
+modprobe tun 2>/dev/null || true
+step "pacman -Syu + tailscale openssh git rsync tmux fish bind (bind = dig, for the DNS checks)"
+pacman -Syu --needed --noconfirm tailscale openssh git rsync tmux fish bind
 step "enable sshd + tailscaled"
-systemctl enable --now sshd tailscaled
+systemctl enable --now sshd
+systemctl enable tailscaled
+if [ ! -d /sys/module/tun ]; then
+  echo "!! The kernel was upgraded and the tun module can't be loaded until you reboot." >&2
+  echo "   Reboot, then rerun fleet-onboard (sshd and tailscaled will start at boot)." >&2
+  exit 1
+fi
+systemctl start tailscaled
 
 # 3. SSH login menu. Source of truth: linux-config home/dot_config/fish/conf.d/ssh-session-menu.fish
 # (private repo, so a fresh box can't fetch it). Embedded copy; if you change the menu there, re-paste it here.
@@ -120,8 +148,9 @@ dest="$home/.config/fish/conf.d/ssh-session-menu.fish"
 as_user() { sudo -u "$user" -- "$@"; }   # writes happen as the user: right owner, user's umask
 if [ "$(basename "$login_shell")" != fish ]; then
   step "ssh menu: skipped, $user's login shell is $login_shell, not fish (chsh -s /usr/bin/fish $user, then rerun)"
-elif [ -L "$dest" ]; then
-  step "ssh menu: $dest is a symlink (stow/chezmoi?), leaving it alone"
+elif [ -L "$dest" ] || [ "$(realpath -m "$(dirname "$dest")")" != "$(realpath -m "$home")/.config/fish/conf.d" ]; then
+  # the file or a parent dir (~/.config, fish, conf.d) is a symlink, e.g. into a dotfiles repo
+  step "ssh menu: $dest or a parent dir is a symlink (stow/chezmoi?), leaving it alone"
 elif [ -f "$dest" ] && [ "$(cat "$dest")" = "$MENU" ]; then
   step "ssh menu: already installed at $dest"
 else
@@ -137,12 +166,12 @@ fi
 stub=/run/systemd/resolve/stub-resolv.conf
 switched=0
 restart_dns() {
-  systemctl restart systemd-resolved
-  if systemctl is-active -q NetworkManager; then systemctl restart NetworkManager; fi
+  systemctl restart systemd-resolved || return 1
+  if systemctl is-active -q NetworkManager; then systemctl restart NetworkManager || return 1; fi
 }
 if ! systemctl is-active -q systemd-resolved; then
   step "systemd-resolved not running: skipped the MagicDNS fix (check 'tailscale status' for a DNS warning)"
-elif [ "$(readlink /etc/resolv.conf)" = "$stub" ]; then
+elif [ "$(readlink -f /etc/resolv.conf)" = "$stub" ]; then   # -f: also matches ../run/… relative links
   if wait_dns; then
     step "resolv.conf already points at the resolved stub, DNS works"
   else
@@ -156,8 +185,9 @@ else
   if ! dns_ok; then
     echo "DNS doesn't work even before the change; not touching resolv.conf. Fix networking first." >&2; exit 1
   fi
-  if [ ! -e "$stub" ]; then
-    echo "$stub missing (DNSStubListener=no?): switching would break DNS; skipped. MagicDNS may not work." >&2
+  if [ ! -e "$stub" ] || ! dig_ok @127.0.0.53; then
+    echo "resolved's stub listener (127.0.0.53) isn't answering (DNSStubListener=no?): switching would break" >&2
+    echo "DNS, so resolv.conf was left alone. MagicDNS may not work; check 'resolvectl status'." >&2
   else
     backup="/etc/resolv.conf.bak-$(date +%F-%H%M%S)"
     cp -a /etc/resolv.conf "$backup"   # set -e: no backup, no change
@@ -182,10 +212,16 @@ fi
 [ "$switched" = 0 ] || systemctl restart tailscaled
 state=""
 for _ in $(seq 1 20); do
-  state="$(tailscale status --json 2>/dev/null | sed -n 's/.*"BackendState": *"\([A-Za-z]*\)".*/\1/p' | head -1)"
+  # `|| true`: a failing status call (tailscaled still starting) must mean "retry", not abort under set -e
+  state="$(tailscale status --json 2>/dev/null | sed -n 's/.*"BackendState": *"\([A-Za-z]*\)".*/\1/p' | head -1 || true)"
   case "$state" in Running|NeedsLogin|NeedsMachineAuth|Stopped) break ;; esac
   sleep 1
 done
+if [ -z "$state" ]; then
+  echo "!! tailscaled isn't answering after 20s. Check 'systemctl status tailscaled' and 'journalctl -u tailscaled'," >&2
+  echo "   fix it, then rerun fleet-onboard." >&2
+  exit 1
+fi
 case "$state" in
   Running) step "tailscale: already logged in" ;;
   NeedsMachineAuth) step "tailscale: logged in, waiting for admin approval in the console" ;;
@@ -199,7 +235,8 @@ case "$state" in
     fi ;;
 esac
 [ -z "$keyfile" ] || rm -f "$keyfile"
-ip4="$(tailscale ip -4 | head -1)"
+ip4="$(tailscale ip -4 2>/dev/null | head -1 || true)"
+ip4="${ip4:-<no IP yet: approve $host in the admin console, then run 'tailscale ip -4'>}"
 
 # 6. hand-off
 echo
diff --git a/fleet/README.md b/fleet/README.md
index 6e96ea6..82a28f1 100644
--- a/fleet/README.md
+++ b/fleet/README.md
@@ -18,27 +18,51 @@ command myself: the laptop and the lab workstations, all on the tailnet `tail2e7
 2. **Repos live in `~/Projects`.** `~/code` is legacy and being emptied (fleet-audit flags it as LEGACY).
 3. **Before deleting a copy**, all of these must hold:
    - a `fleet-audit --fetch --all` run ends with every host `audited` and exits 0. A host marked
-     `NOT AUDITED` or `INCOMPLETE` proves nothing, because a missing row is not a clean row;
-   - the copy's row is **present** and shows `ok`, or only info flags (BEHIND, UNLISTED/MISPLACED,
-     LEGACY). Any of NOGIT, NESTED, NOREMOTE, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, IGNORED,
-     WORKTREES, SUBMODULES, LOCALREMOTE, FETCHFAIL or ERR means it holds something not saved elsewhere;
-   - for `IGNORED:n` (usually the data), the ignored files are copied to the owner host and verified as
-     in "Moving gitignored data" below. Git can't vouch for them.
-   Deleting a whole root (e.g. `~/code`) also needs no `LOOSE:n` row for it.
+     `NOT AUDITED` or `INCOMPLETE` proves nothing, because a missing row is not a clean row. (A
+     `skipped host` line is fine only if that host isn't the one you're deleting from.)
+   - the copy's row is **present** and shows `ok`, or only these info flags: BEHIND, MISPLACED,
+     UNLISTED, LEGACY, EMPTY (an empty dir: `rmdir` it). `UNDECIDED` means decide the owner first.
+   - **Stop flags.** Any of these means the copy holds something not saved elsewhere: NOGIT, NESTED,
+     NOREMOTE, LOCALREMOTE, SUBMODULE-NOBACKUP, UNPUSHED, DETACHED, DIRTY, UNTRACKED, STASH, HIDDENIDX,
+     LFS, IGNORED, WORKTREES, SUBMODULES, FETCHFAIL, ERR. Resolve each one, then re-run:
+     - `IGNORED:n` (usually the data): copy and verify it as in "Moving gitignored data" below.
+       Git can't vouch for it.
+     - `LFS:n`: run `git lfs push --all <forge-remote>` (it uploads only what the server lacks) and
+       check that it exits 0, or keep the copy.
+     - `HIDDENIDX:n`: `git ls-files -v | grep -E '^([a-z]|S) '` lists the files. Clear the bit
+       (`git update-index --no-assume-unchanged / --no-skip-worktree <f>`) and commit any real edit.
+   - Deleting a whole root (e.g. `~/code`) also needs no `LOOSE:n` row for it.
+
+   **What counts as "saved elsewhere":** only a remote on a forge (github.com, gitlab.com,
+   git.overleaf.com, bitbucket.org, codeberg.org), checked by both its configured and its rewritten
+   URL. A copy on another fleet box (`user@lab:…`, `ssh://lab/…`) or a local path is not a backup.
+
+   **Not counted as rebuildable** (so they show up as IGNORED): `.ipynb_checkpoints`, `.cache`,
+   `env/`. Only `.venv`, `venv`, `__pycache__`, `node_modules`, pytest/mypy/ruff caches, `.tox`/`.nox`
+   and `*.egg-info` are skipped.
+
+   **Out of scope**, so deleting a copy loses these:
+   - reflog-only commits (work you already reset or amended away);
+   - annotated tag objects you never pushed, when they point at a commit that *is* pushed;
+   - git-annex content.
+
+   If any of that matters for a repo, keep a full copy (`tar` or `git bundle create x.bundle --all`).
 
 ## Moving gitignored data to the owner host
 
 `rsync --ignore-existing` + "compare sizes" is NOT enough: it keeps an older, different file at the
 destination, and equal sizes don't mean equal contents. Instead:
 ```bash
+# 0. stop anything still writing to the data
 # 1. copy into a separate staging dir, so nothing on the destination is overwritten
 rsync -aP <repo>/data/ lab:Projects/<repo>/_incoming-from-laptop/data/
-# 2. verify by checksum: prints nothing when every file arrived identical
-rsync -anci <repo>/data/ lab:Projects/<repo>/_incoming-from-laptop/data/
+# 2. verify by checksum. Success = exit status 0 AND no output. Any listed file differs: re-run 1, then 2.
+rsync -anci <repo>/data/ lab:Projects/<repo>/_incoming-from-laptop/data/; echo "exit=$?"
 # 3. on the owner host, merge the staging dir by hand. Where both sides have a file and they differ,
 #    decide which to keep; don't let a tool silently pick.
 ```
-Stop anything still writing to the data first. Only then is the source copy deletable.
+Until it's merged, `_incoming-from-laptop/` shows as UNTRACKED on the owner host. That's the reminder.
+Only after step 2 passes is the source copy's data safe to delete.
 
 ## Adding a lab machine (CachyOS already installed)
 
@@ -60,10 +84,13 @@ Never put the key on the command line or in an env var you type: it would land i
 the process list. `--authkey` reads it hidden and hands it to tailscale through a 0600 temp file.
 
 It runs `pacman -Syu` (full upgrade, so openssh isn't half-upgraded), installs and enables `sshd` +
-`tailscaled`, applies the CachyOS MagicDNS fix (the resolv.conf symlink), and joins the tailnet. The
-DNS change is guarded: it only switches if DNS works beforehand and the resolved stub exists, keeps a
-timestamped `/etc/resolv.conf.bak-*`, and restores it (exit 1) if DNS breaks afterwards. It refuses
-to run over SSH (the restarts can cut the session) unless you pass `--force`. It leaves password login on. It also installs the SSH
+`tailscaled` (plus `bind`, for `dig`), applies the CachyOS MagicDNS fix (the resolv.conf symlink),
+and joins the tailnet. If the upgrade installed a new kernel, tailscale's `tun` module can't load
+until a reboot. It then says so and exits: reboot and run it again. The DNS change is guarded. It
+only switches if DNS works beforehand and resolved's stub listener answers, and it keeps a timestamped
+`/etc/resolv.conf.bak-*`. Afterwards it checks DNS through `/etc/resolv.conf` itself and restores the
+backup (exit 1) if that broke. It refuses to run over SSH, which it detects even through sudo, because
+the restarts can cut the session. `--force` overrides that. It leaves password login on. It also installs the SSH
 login menu (1 plain fish / 2 tmux / 3 herdr, plain fish after 5 s) into `~/.config/fish/conf.d/`, but only
 if your login shell is fish. That menu is an embedded copy of
 `linux-config/home/dot_config/fish/conf.d/ssh-session-menu.fish`, so if you change the menu there,
```
