# Review of `fleet` commit 7594a87 (round 4: the round-3 fixes)

**Reviewer:** Claude Fable 5.1 (model id `claude-fable-5-1`), Anthropic.

**Method.** I read `bin/fleet-audit`, `bin/fleet-onboard` and `fleet/README.md` in full at 7594a87, then
built a throwaway fleet on this machine (git 2.56, GNU awk 5.4, git-lfs 3.8): a local "forge" of bare
repos reached through the `FLEET_AUDIT_FORGES` / `FLEET_AUDIT_GIT_SSH` test hooks with a fake `ssh`
that runs `git-upload-pack` locally, 15 scenario repos under an `ssh: local` host, plus 22 `hosts.yaml`
variants. Everything below marked "tested" was actually run; `fleet-onboard` was reviewed statically
(it self-elevates and upgrades the box, so I did not execute it).

## Round-3 items

| Item | Verdict | What I tried |
|---|---|---|
| C1 hosts.yaml dropped hosts | **FIXED** | 22 variants: host id at column 0, tab-indented stanza, tab inside a value, tab only in an inline comment (accepted, correct), stray top-level key, `---` marker, content before `hosts:`, no `hosts:`, empty file, UTF-8 BOM, second `hosts:`, key before any host, 3-/6-space keys, `Ssh:`, quoted id, id with space, YAML list for roots, CRLF. Every malformed one exits 2 with a line number before any scan; the two valid ones parse both hosts. |
| C2 unreadable dir in a repo | **FIXED** | `chmod 000 locked/` with a file inside: `git status --porcelain` exits 0 and prints `warning: could not open directory` on stderr. Row shows `ERR`; the host summary prints the warning under `!! scanner warnings`. Gitignored unreadable dir gives `!!` + IGNORED instead (no warning needed). |
| C3 inaccessible parent → ROOT-MISSING | **FIXED** | Root under a `chmod 000` parent → `#ROOTERR`, host `INCOMPLETE`, exit 1. Dangling-symlink root → same. Truly absent root → `ROOT-MISSING`. coreutils message is `cannot statx '…': No such file or directory`, matched. |
| C4 stale tracking refs counted as pushed | **FIXED** (one misdiagnosis, see N1) | Narrowed refspec `+refs/heads/main:…`, branch `old` deleted on the server, `origin/old` left behind: without `--fetch` the row is clean (documented), with `--fetch` → `UNPUSHED:1`. Server force-pushed back one commit → `UNPUSHED:1`. Commit reachable only via a pushed annotated tag (`^{}` peel line) → clean, correct. Second forge remote unreachable → `FETCHFAIL`. Empty server repo → `ERR` instead of `UNPUSHED:n` (N1). 3000 `^sha` lines on stdin: 2 ms. |
| C5 skip-worktree names needing quoting | **FIXED** | File named `ta<TAB>b` with skip-worktree set → `HIDDENIDX:1`. `-z` output is `S␠path␀`, `${rec:2}` is right. |
| C6 LFS: `lfs.storage` ignored, unreadable store = 0 | **PARTIAL** | Absolute `lfs.storage` with 2 objects → `LFS:2`; `~/lfsx` → `LFS:1`; a `find` failure → ERR (by code). **But** `chmod 000 .git/lfs` with an object inside → row is clean (`UNLISTED` only). See B1. |
| C7 replace refs | **FIXED** | `git replace <blob> <otherblob>` → `REPLACE:1`. README instruction to clear it is wrong (N3). |
| C8 README `git bundle --all` | **FIXED** (doc nits N4) | tar command works from `<parent>`; run from inside the repo it exits 1 (`file changed as we read it`) and leaves the partial tarball inside the repo. Unreadable dir inside → exit 2, no `OK`. Good. |
| N3/N5 empty forge host; push URLs | **FIXED** | `url.ssh://lab/x/.pushInsteadOf ssh://forge/` is visible through `remote get-url --push --all` → remote not a backup. `is_forge` rejects an empty host. |
| N2/N4 README LFS/IGNORED wording | **PARTIAL** | LFS/IGNORED text is now right about what clears them. The new "see the actual commits" command contradicts the `--fetch` result (N2). |
| fleet-onboard N6/N7 | **FIXED** (static) | Key prompt now after `pacman -Syu`, the tun check and `systemctl start tailscaled`; `tun_ok` accepts `CONFIG_TUN=y` from `/proc/config.gz` (present on CachyOS; here `CONFIG_TUN=m` and `/sys/module/tun` exists, so the fallback is dormant and harmless). |

## Blocking

### B1 · MAJOR (borderline, so listed here) · `bin/fleet-audit:267` — an LFS store that exists but can't be traversed is reported as 0 objects, and the row comes out clean

**Class:** false "clean" under the README rule.

`count_lfs` does `[ -e "$st/objects" ] || return 0`. `-e` is false not only for ENOENT but for EACCES on
`$st` itself. The `find`-exit-status fix only runs after that test, so it never sees this case.

**Trigger (reproduced):** repo `r5b`, pushed and otherwise clean, with `.git/lfs/objects/ab/1` present
and `.git/lfs` mode 000 (equivalently: `.git/lfs` created by root with a 077 umask, e.g. `sudo git lfs
fetch` in a repo that had no `.git/lfs` yet). `fleet-audit --fetch --all` prints
`r5b … UNLISTED` with no `LFS`, no `ERR`, host `audited`, exit 0. By the README rule that copy is
deletable; the LFS object is lost. Same outcome if `$st` is a dangling symlink or any parent of it is
non-traversable.

Why I put it under Blocking despite the low likelihood: it is exactly the "unreadable store counts as
0" hole round 3 reported as C6, the fix claims to close it, and the fix is three lines. Why it is
borderline: on CachyOS the root umask is 022, so a 700 root-owned `.git/lfs` needs a non-default
setup or a manual chmod.

**Fix:** treat only ENOENT as "no store", like the roots do:
```bash
if [ ! -e "$st/objects" ]; then
  msg=$(LC_ALL=C stat -- "$st/objects" 2>&1 >/dev/null)
  [[ "$msg" == *"No such file or directory"* ]] && [ ! -L "$st" ] && return 0
  return 1    # EACCES / ENOTDIR / dangling symlink -> ERR
fi
```
(`stat` on `.git/lfs/objects` with `.git/lfs` mode 000 says `Permission denied`; with `.git/lfs`
absent it says `No such file or directory`, which is the case where 0 is right.)

## Non-blocking

### N1 · MAJOR · `bin/fleet-audit:228-231` — empty server repo gives `ERR` instead of `UNPUSHED:n`, and a repo-less warning
When `ls-remote` succeeds with an empty advertisement (fresh GitHub repo, nothing pushed yet),
`cut -f1 <<<""` emits one empty line, `cat-file --batch-check` answers `␠missing`, awk sees
`$1=missing $2=""`, prints `^missing`, and `rev-list` dies with `fatal: bad revision '^missing'`.
Tested: repo `r3` shows `ERR` and the host summary prints `fatal: bad revision '^missing'` with no
repo name. Doesn't block: ERR is a stop flag, so the direction is safe. But it misdiagnoses the most
common real case of "I created the repo on GitHub and never pushed", and it puts a repo-less fatal
line in every run. Fix: `cut -f1 <<<"$adv" | awk 'NF' | sort -u | …` and `awk 'NF==2 && $2!="missing"'`;
prefix the rev-list stderr with `$d` like `count_status` does.

### N2 · MAJOR (doc) · `fleet/README.md:150-153` — the "see the actual commits" command contradicts the tool
The README tells the owner to run `git log … --all --not --remotes=origin` "before assuming work is at
risk". That uses cached tracking refs, which is precisely the evidence C4 removed. Tested on `r2`: the
audit says `UNPUSHED:1`, the README command prints nothing. An owner following it concludes the tool
is wrong. Doesn't block: the deletion rule still requires the row to be `ok`, so the tool wins. Fix:
```bash
git ls-remote origin | cut -f1 | awk 'NF {print "^" $1}' \
  | git log --oneline --exclude='refs/remotes/*' --exclude=refs/stash --all --stdin
```
(and drop or qualify "a squash-merged branch shows up here too": with `--fetch`, if a PR existed, GitHub
still advertises `refs/pull/N/head`, so those commits correctly count as on the forge).

### N3 · MINOR (doc) · `fleet/README.md:38-39` — `git replace -d <ref>` doesn't work
`for-each-ref refs/replace` prints `refs/replace/<sha>`; `git replace -d refs/replace/<sha>` fails
(`replace ref '…' not found`, tested). It needs the bare `<sha>` (the replaced object). Say
`git replace -d <sha>` or `git replace -l` to list. Not blocking: the flag stays until the ref is gone.

### N4 · MINOR (doc) · `fleet/README.md:55-61` — the tar recipe assumes CWD = parent and says nothing about where the archive goes
Run from inside the repo it exits 1 (`file changed as we read it`) and leaves `<repo>-DATE.tar.gz`
inside the repo (tested). Run from `~/Projects` it leaves the tarball in a root, which `fleet-audit`
will then flag as `LOOSE:1` (fine, but say so). Also `tar -tzf` proves the archive is readable, not that
it matches disk; `tar -df` (`--diff`) from `<parent>` does. Suggest: `cd <parent>` first, write the
archive outside every root (or straight to the owner host), verify with `tar -df`. Not blocking:
all failure modes fail loudly.

### N5 · MINOR (doc) · `fleet/README.md:32-35` — the LFS clearance condition lost its fallback
`git lfs push --all` pushes objects referenced by commits reachable from local refs (man page checked);
objects in the store that no local ref references are not pushed, which is consistent with the
"reflog-only is out of scope" note, fine. But the old text's "or keep the copy" is gone: if the push
exits non-zero (typical: pointer whose object was never downloaded, `Unable to find source`), the
reader has no instruction. Add "non-zero: `git lfs fetch --all` then retry, or keep the copy".

### N6 · MINOR · `bin/fleet-audit:265-266` — `lfs.storage` edge cases
A *global* absolute `lfs.storage` (a shared store, which is how git-lfs documents it) makes every
repo report the whole store's count (`LFS:n` everywhere): noisy, not unsafe. `~user/x` becomes
`$HOME/user/x`. The owner's laptop has no global `lfs.storage`, so this is dormant.

### N7 · NIT · `bin/fleet-onboard:106-112` — key typo is now detected only after the full upgrade
Moving the prompt after `pacman -Syu` (correct for the reboot case) means a mistyped key costs the
owner a rerun after a possibly long upgrade; validating the format up front and *re-asking* after the
tun check would keep both properties. Also "the rest runs unattended" is only true once the prompt has
appeared, so the operator has to stay until the upgrade ends. And if TUN is genuinely absent from a
kernel, `tun_ok`'s advice ("reboot, then rerun") loops forever; worth one sentence.

### N8 · NIT · `bin/fleet-audit:158,226-229,222` — small robustness items in the new code
`sed "s#^#$d: #"` breaks if `$d` contains `#` (and `&` is substituted); use `awk -v d="$d" '{print d": "$0}'`.
The `cut | sort | cat-file | awk >>tf` pipeline's status is never checked; every failure shrinks the
negation set, so the direction is conservative, but a `|| rc=1` would make it visible. The scanner has
no `trap` for its mktemp files, so a `timeout` kill leaves them in the remote's `/tmp`.

### N9 · NIT · `bin/fleet-audit:456-460` — "scanner warnings … the affected repos show ERR" over-promises on ssh hosts
`$tmp/$id.err` also receives ssh's own stderr (known_hosts notices, banners), which is then printed
under a heading that says the affected repos show ERR. Cosmetic.

## What I checked and found correct

- **`rev-list --count --exclude=… --all --stdin` with `^sha` lines**: the `--exclude` patterns are
  consumed by `--all` before stdin is read, stdin negations are plain revisions, so semantics are
  "commits reachable from local refs minus everything reachable from any advertised SHA". Peeled
  `^{}` lines negate the commit directly and the tag object peels too (tested). Advertised refs that
  point at blobs/trees are silently ignored by rev-list without `--objects` (tested with `^<blob>`
  and `^<tree>`, exit 0). Objects reported `missing` by `cat-file` are dropped, which only grows the
  count. A corrupt object either reports `missing` (dropped) or makes `rev-list` fatal (ERR). Both safe.
- **Filtering to locally present objects** is sound because `fetch --prune <remote>` runs first; a
  server commit we lack (narrowed refspec, `refs/pull/*`) is dropped and its local ancestors are
  over-reported, never under-reported.
- **Partial ls-remote failure** (one of two forge remotes down): `FETCHFAIL` plus a count built from the
  remaining remotes, which is a superset of the truth (tested, `r12`).
- **`count_unpushed` return codes**: `rc` is `local` in `scan_repo`; 1 → ERR, 2 → FETCHFAIL, both for the
  repo and for submodules; temp file removed on every path.
- **`count_status`**: `2>"$tf.err"` captures the warning in porcelain mode (verified wording), rc forced
  to 1, both temp files removed. Submodule with `--ignore-submodules=none` and an uninitialised
  submodule produce no stderr, so no spurious ERR (tested).
- **`count_hidden`**: explicit `rc`, `-e || -L` test, correct `${rec:2}` offset for `-z`.
- **`count_replace`**: `for-each-ref … refs/replace/` prefix match, pipefail propagates a failure to ERR.
- **`backup_remotes`**: `get-url --push --all` surfaces `pushurl` and `pushInsteadOf`; `is_forge`
  rejects empty hosts; URL host extraction handles `user:pass@`, `host@evil`, scp-style, and paths.
- **hosts.yaml parser**: every malformed shape exits 2 before any host is contacted; the `# comment`
  and trailing-whitespace normalisation happens before the tab check, so a tab inside a comment is
  accepted (correct).
- **Roots**: EACCES-on-parent and dangling symlink both mark the host INCOMPLETE (exit 1); only a true
  ENOENT yields ROOT-MISSING. The `stat` message is captured in a variable, so pipefail is not involved.
- **fleet-onboard**: nothing in the diff adds a path that touches sshd auth, resolv.conf or
  NetworkManager; the key read happens with `tailscaled` already started and before anything
  network-affecting, and the EXIT trap still deletes the key file. `exit 1` at the tun check now costs no
  typed key. `zgrep -qx 'CONFIG_TUN=y'` is a correct whole-line match; `zgrep` ships with `gzip` (base).

## Bottom line

1. **fleet-onboard safe for the 3 machines?** Yes. The round-4 changes are confined to prompt order and
   a looser tun check; neither can cut the network or lock the owner out, and idempotency is unchanged.
   N7 is a convenience nit.
2. **fleet-audit trustworthy for deletion under the README rule?** Almost. Every round-3 false-clean path
   I could reproduce is closed except one: an LFS store directory that exists but cannot be traversed is
   counted as 0 and the row comes out clean (B1). It needs a non-default permission state, but it is the
   exact class the C6 fix claims to close and the fix is three lines. The `--fetch` UNPUSHED redesign
   is correct and conservative in every scenario I tried.
3. **Ready to merge?** Not as-is: fix B1 first (and ideally N1, which will otherwise show `ERR` on every
   freshly created, never-pushed GitHub repo). With B1 fixed I would merge; N2 to N5 are README
   corrections that should follow quickly because the README is the deletion procedure.
