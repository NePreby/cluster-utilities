# Independent review of c3eeb4d

Model: **GPT-6 (Codex)**. No more specific model version identifier is available to me.

Scope: the round-5 changes and their interactions with the deletion rule. I read `bin/fleet-audit` and `fleet/README.md` in full, plus this round's `REVIEW_REQUEST.md`. I did not read any other file under `reviews/`. This workspace is a source snapshot without Git metadata, so the supplied diff defines the change boundary; the findings below refer to the actual files on disk.

I exercised the actual scanner, extracted functions, and an unchanged copy of the complete executable in isolated temporary fixtures outside the repository. Full-tool tests used real local Git repositories and a local transport shim with the documented forge test hooks; they did not contact the real fleet or GitHub. SSH cancellation was additionally tested against a real, temporary loopback sshd. Repository changes are confined to this review.

## Round-4 verification

| Item | Verdict | Concrete input tried and result |
|---|---|---|
| D1 — inaccessible / dangling / symlinked LFS stores | **FIXED** | With an unprivileged user, `chmod 000` on `.git/lfs`, a custom storage directory, or its `objects` directory returned failure rather than zero objects. A dangling `objects` symlink returned failure. An `objects` symlink to a readable directory containing one file produced `LFS:1` in the complete audit. A regular file in place of the storage directory returned failure. |
| D2 — a successful LFS push must not clear LFS | **PARTIAL** | The push-based exception is removed. However, both an absolute custom store and a symlinked `objects` store can produce `LFS:1` while the replacement archive recipe prints `VERIFIED` without preserving that content. See B1. |
| D3 — ENOENT suffix and root validation | **FIXED** | Tested a missing path, ENOTDIR, ELOOP, an inaccessible parent named `denied: No such file or directory`, and that suffix at the end of the tested pathname. Only definite absence returned success. A relative root and `/tmp/.../My Projects` failed inventory parsing with exit 2; absolute and `~/...` roots passed. Non-English locale behavior was inspected, not validated under an installed non-English locale: this environment has only C and English locales; `stat` is explicitly run with `LC_ALL=C`. |
| N1 — empty advertisement | **FIXED** | A one-commit repository pointing at an empty bare server produced `UNPUSHED:1`, without the former spurious ERR. Pushing that commit changed the same fixture to `ok`. |
| N2 — cat-file failure / stderr | **FIXED** | Fault-injected `cat-file` exit 42 and, separately, successful real output plus a warning. The complete audit produced `UNPUSHED:1 ERR` and `ERR`, respectively. The warning was displayed. Successful output with repeated advertised SHAs worked normally. |
| N3 — manual command uses current advertisement | **FIXED** | Created a unique second local commit and a stale `refs/remotes/origin/stale` pointing to it, while the server advertised only the first commit. The README pipeline printed the unique second commit. |
| N4 — deleting replace refs | **FIXED** | Created a real blob replacement, used the original object SHA printed by `git replace -l` with `git replace -d`, and confirmed the replace list became empty. |
| N5 — tar location and verification | **PARTIAL** | A stationary repository with an ordinary internal LFS store archived and compared successfully. External stores were omitted despite `VERIFIED` (B1). Two same-named copies archived on the same day overwrote one another in the new shared archive directory (B2). |
| N6 — temporary directory, traps, warning prefix | **PARTIAL** | Normal exit, local timeout, and individual TERM/INT/HUP signals removed scanner temporaries; interrupted scanners did not emit `#DONE`. A warning from `warning#repo/denied` retained its pathname prefix and set ERR. A real SSH timeout left the remote temporary directory present after the client returned and three seconds later; it disappeared after the remote scan eventually finished. See N1 below. |
| N7 — tab wording | **FIXED** | Tabs in indentation and `ssh:` content caused exit 2; trailing tabs and tabs in stripped comments were accepted, matching the revised text. |

## Blocking

### B1 — BLOCKER: the new LFS archive exception does not preserve external storage

**Class: false clearance under the README deletion rule, leading to data loss.**

**Locations:** `fleet/README.md:31` and `fleet/README.md:56`–65; interaction with `bin/fleet-audit:285`–293.

The scanner explicitly supports custom `lfs.storage` and now follows an `objects` symlink. The replacement deletion instructions nevertheless archive only the checkout directory. GNU tar does not follow symlinks by default, and an absolute custom store is not even an archive operand. `tar -d` can successfully compare everything that was archived while the actual LFS payload is absent.

**Concrete reproduction, including a deletion path that loses the data:**

1. Create two otherwise backed-up checkouts, `root/repo` and `root/shared`.
2. Put the only copy of an LFS object in `root/shared/.git/lfs-cache/objects/unique`. Configure `repo` with absolute `lfs.storage` pointing to `shared/.git/lfs-cache`. Alternatively, make `repo/.git/lfs/objects` a symlink to that cache's `objects` directory.
3. Run `fleet-audit --fetch --all`. My complete-tool result was:

   ```text
   repo    ... LFS:1
   shared  ... BEHIND:1
   == test: audited (2 entries)
   ```

   Exit status was 0. `BEHIND` is expressly allowed for deletion. `shared` does not use this cache as its own LFS store, and Git status does not report arbitrary files inside `.git`.
4. Follow the README recipe for `repo`, and copy its archive off-machine. Creation and comparison returned 0 and printed `VERIFIED`; extracting all archived file contents confirmed the unique payload was absent.
5. Delete both checkouts using their permitted rows and the documented LFS exception. Deleting `shared` destroys the only object; the supposedly verified backup of `repo` cannot restore it.

This is an interaction with the newly prescribed LFS remedy, not a request to audit arbitrary `.git` contents. Even without the sibling-checkout example, the archive is not the claimed independent backup of an external store.

**Suggested fix:** Do not let this recipe clear LFS until the actual storage location has been resolved and its contents are included and verified. The minimal conservative fix is to keep LFS blocking for custom/external or symlinked storage until that store has a separate verified backup. A general recipe needs to archive the actual Git directory and resolved LFS storage, preserve enough location information to restore them, and verify payload contents. Merely comparing a symlink is insufficient.

### B2 — MAJOR: the new shared archive path silently replaces a previous copy's backup

**Class: data loss through the new archive-based deletion procedure.**

**Location:** `fleet/README.md:60`–65, especially the filename assignment on line 61.

Every checkout named `<repo>` now writes to the same `~/archives/<repo>-<date>.tar.gz`, regardless of its original root or host. The documented migration specifically involves same-named copies under `~/code` and `~/Projects`. `tar -c` truncates an existing archive; successful `tar -d` verifies the replacement, not preservation of the earlier backup.

**Concrete trigger:** Archive `~/code/repo`, whose store contains only `ONLY-IN-LEGACY-COPY`; copy that archive into an off-machine backup directory and delete the source after verification. Later the same day archive `~/Projects/repo`, whose store contains only `ONLY-IN-CURRENT-COPY`, and copy it into the same backup directory using an ordinary overwriting copy. Both operations follow the recipe and print `VERIFIED`. Both archive destinations now contain only the second payload.

I reproduced both successful comparisons, the local overwrite, and the overwrite in a separate backup destination. Extracting `repo/.git/lfs/objects/object` from the surviving backup returned only `ONLY-IN-CURRENT-COPY`.

The change to a single prescribed archive directory makes collisions between these different roots predictable. The new LFS exception makes preservation of these archives a prerequisite for safe deletion.

**Suggested fix:** Allocate a fresh archive destination for each source copy, with collision-resistant identity and no silent reuse—e.g. a newly created unique directory containing host/root metadata and the archive. Preserve that unique identity when copying off-machine, and prohibit replacement of an existing backup. Verify the destination copy before deleting the source.

## Non-blocking

### N1 — MINOR: local SSH timeout does not guarantee remote scanner cleanup at that time

**Locations:** `bin/fleet-audit:137`–139 and `bin/fleet-audit:447`.

The new comment promises cleanup when `timeout` kills the scan, but on the SSH path the timed process is the local SSH client. Disconnecting a noninteractive SSH session does not reliably deliver TERM, INT, or HUP to the remote Bash process. Its EXIT trap runs when that remote process actually exits.

**Concrete trigger:** With a real loopback sshd, run the actual scanner through `timeout 0.5 ssh ... bash -s ...`, with a Git wrapper that sleeps two seconds before each real command. The client returned 124 with no `#DONE`. The remote `SCANTMP` directory still existed immediately and three seconds later, and disappeared only after the scan finished. A hung foreground command could extend that lifetime substantially.

**Why this does not block:** The local SSH command's nonzero status prevents the host being accepted as audited, independently of `#DONE`. The tested local timeout also made the complete tool exit 1 with `NOT AUDITED`. This is a cleanup/runtime guarantee issue, not a false clean.

**Suggested fix:** Either qualify the cleanup comment to describe this limitation, or enforce the scan deadline on the remote side too, with enough allowance in the outer SSH timeout for the remote trap to finish.

## What checked out

- The tested D1 changes fail closed for permission errors, ENOTDIR, ELOOP, and dangling terminal symlinks. Readable symlinked stores are counted.
- `absent()` examines the diagnostic suffix rather than a pathname substring. A successful `stat` after an existence-check race would yield an empty captured diagnostic and would not be classified as absent.
- Empty advertisements no longer generate a fake revision. Fresh advertised tips, rather than stale remote-tracking refs, determine exclusions in the tested cases.
- Both `cat-file` failure and warning paths set the repository's ERR stop flag. Exit 0 for the overall host in these tests is consistent with the README: an audited host is not sufficient; its row must also pass.
- Local cancellation runs the cleanup trap and suppresses `#DONE`. The parent requires both exit 0 and the terminal marker, so the tested timeouts cannot masquerade as successful audits.
- The revised replacement-ref command, strict root-prefix checks, and tab description matched the tested behavior.
- The tar recipe works for the ordinary internal-store, stationary-source case; its failure is the broader safety claim and backup identity, not basic gzip/archive syntax.

## Prioritized bottom line

1. **Fix B1 before deletion:** the documented LFS exception currently permits deleting unique content that the verified archive never contained.
2. **Fix B2 before relying on these archives:** distinct copies must not overwrite each other's only backups.
3. N1 can follow separately; timeout failure remains conservative.

**(a) Is fleet-audit trustworthy for deletion under the README rule? No.** The tested scanner fixes are conservative, but the rule's new archive exceptions are not safe as written.

**(b) Ready to merge into main? No.** Resolve B1 and B2 first.
