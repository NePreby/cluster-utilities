# Independent review of e5acf77

Model: GPT-6 (Codex). A more specific model version identifier is not exposed to me.

Scope: the changes described in `REVIEW_REQUEST.md`, relative to round 6. I read the actual `bin/fleet-audit` and `fleet/README.md` in full, plus the inventory, placement configuration, and `AGENT.md`. I read no other review. This checkout has no `.git` directory, so I could not independently verify the commit identity or historical diff; the supplied diff defines the change boundary.

Tests used isolated temporary fixtures, the actual embedded scanner extracted from `bin/fleet-audit`, and the actual README command chains with placeholders substituted. Archive and checksum tests used real GNU tar, find, sha256sum, and rsync. SSH transport was emulated locally by executing its remote command under fish 4.9.3 in a separate destination directory; no real fleet host was contacted. Only this report was changed in the repository.

| Round-6 item | Verdict | Concrete input tried and result |
|---|---|---|
| F1: symlinks below the LFS objects directory | **PARTIAL overall; scanner FIXED for tested static inputs** | An all-linked shard, an all-linked object, and a mixed regular/linked store produce `LFS:1 LFS-EXTERNAL`, `LFS:1 LFS-EXTERNAL`, and `LFS:2 LFS-EXTERNAL`, respectively. A dangling child or link to an empty directory produces `LFS-EXTERNAL` even with zero regular objects. A loop produces `ERR`. However, the new external-store archive exception still approves backups lacking the objects: B1. |
| F2: approve a different remote archive | **PARTIAL** | An already-existing destination containing different bytes and its own valid checksum is refused and left unchanged. Altered received bytes also fail verification. But an archive arriving between the existence check and rsync is overwritten, and the chain prints `COPY OK`: B2. |

## Blocking

### B1 — BLOCKER: The new external-store exception still backs up only links

**Location:** `fleet/README.md:36–38`, invoking the archive recipe at `fleet/README.md:73–85`.

**Why this blocks:** this newly permitted resolution of a deletion stop flag can certify an archive without the sole LFS payload. It is an unsafe deletion procedure, even though the scanner now correctly raises the flag.

Concrete tested layout:

```text
source/repo/                         Git repository
  .git/config                       lfs.storage = /.../source/store
source/store/objects/aa -> /.../external/
external/object                     irreplaceable LFS payload
```

I followed the new instruction literally: archive `repo` from its parent, then archive the configured `store` directory from its parent, using the same tar/create/compare/hash/copy recipe for each. Both runs printed **`VERIFIED` and `COPY OK`**. Inspection of both received archives found **no regular-file member containing the LFS payload**. The store archive contains `store/objects/aa` as a symlink. `tar -d` successfully compares that link; it does not establish that its referent was archived.

The same problem applies when the store directory itself is a symlink, or when individual objects are symlinks. Resolving just the top-level store path does not fix linked descendants. The original external payload may remain on the source host after deleting the checkout, but it still has no verified backup; losing or clearing that storage makes these supposedly sufficient archives unrestorable.

**Suggested fix:** remove the new archive exception and retain the stop/keep-copy requirement until there is a correct procedure. If retaining the exception, require archiving the actual bytes behind every external link, including linked descendants, with all relevant writers stopped. Verify the captured payloads independently of the original targets before declaring `LFS-EXTERNAL` resolved. Repeating the existing non-dereferencing tar recipe on the configured store is insufficient.

### B2 — MAJOR: The remote existence check is not an atomic reservation, and rsync can destroy another backup

**Location:** `fleet/README.md:79–82`.

**Why this blocks:** loss of an existing backup, introduced by allowing ordinary overwriting rsync after a separate check. This is a concurrency-dependent case, but it belongs under Blocking because another copy may already have been deleted based on the backup that gets overwritten.

Concrete trigger: two archive transfers have the same destination basename. The original F2 already establishes that names generated on separate machines need not be globally unique. Schedule them as follows:

1. Transfer A runs `test ! -e archives/$b` successfully.
2. Before A's rsync starts, transfer B creates that directory, transfers its different archive, and verifies it. B can now legitimately treat its backup as complete.
3. A runs `rsync -a "$d" host:archives/`. Unlike the old `--ignore-existing` command, this may replace B's archive.
4. A's remote hash matches A's local hash, so A prints `COPY OK`. B's backup is gone.

I reproduced this deterministically by making the emulated SSH transport create B's destination and valid checksum immediately after the successful existence check, before returning to A. A's local `repo.tar.gz` had different content and an older modification time, ensuring rsync transferred it. The unmodified copy chain printed:

```text
COPY OK: fixture-host:archives/host-repo-date-ABC123
```

The destination previously holding `archive B contents` then held `archive A contents`. The test uses sentinel archive-file contents to isolate the overwrite; rsync and the checksum chain do not depend on the file's archive format. No checksum collision or hostile remote is involved. Stopping writers to A's repository does not stop another archive transfer.

**Suggested fix:** reserve the final destination atomically with `mkdir`, and proceed only if that creation succeeds. For example, after ensuring `archives` exists, require `mkdir archives/$b`, then copy the source directory's *contents* (`"$d/"`) into that reserved directory. A failed or interrupted attempt should leave a reserved destination requiring explicit recovery or a new name. Retain the comparison against the originating machine's checksum.

## Non-blocking

### N1 — NIT: The find comment incorrectly says dangling links cause failure

**Location:** `bin/fleet-audit:303–304`.

**Trigger and observed behavior:** a store containing only `objects/link -> /nonexistent` makes `find -L ... -type f` succeed with zero matches. The second traversal detects the link, and the scanner emits `LFS-EXTERNAL`, not `ERR`.

**Why it does not block:** the actual implementation remains fail-closed for deletion because `LFS-EXTERNAL` is a stop flag. The defect is the explanatory comment.

**Suggested fix:** distinguish loops/traversal failures, which cause `ERR`, from dangling descendants, which the subsequent symlink check flags as external. A dangling `objects` directory itself already produces `ERR` through the earlier existence check.

## What I checked and found correct

- **LFS scanner:** real extracted scanner runs covered regular objects, symlinked shard directories, symlinked object files, mixed stores, a symlinked `objects` directory, dangling children, an empty linked directory, a dangling `objects` directory, and a looping child. Every linked/non-readable case tested retained a stop flag; regular objects retained `LFS:n`. The independent `lfsext` emission at `bin/fleet-audit:398` correctly preserves the flag when the object count is zero.
- **Link containment:** `bin/fleet-audit:312–316` now checks stores containing only links. Internal descendant links are conservatively flagged too, which can require extra manual work but does not produce false deletion permission.
- **Copy checks outside the race:** an existing directory with a different self-consistent archive is refused without alteration. A fresh transfer succeeds through fish. Changed destination bytes, a remote checksum command returning 127, and rsync exiting 23 all print `copy NOT verified` and never `COPY OK`. The `&& ... || echo` grouping works for those cases. Its failure echo returns status zero, so callers must follow the documented marker rule rather than infer success from the chain's exit status.
- **Same-machine destination:** mapping the remote directory to the originating account's archive location causes the existence check to reject the copy. Different accounts or directories are not proof of another physical machine; the README explicitly requires another machine, so I do not treat violating that prerequisite as a new defect.
- **Driver temporary-directory failure:** running the actual driver with `TMPDIR` pointing at a nonexistent directory exits **2**, prints `fleet-audit: mktemp -d failed`, and produces no audit table or host connections (`bin/fleet-audit:455`).
- **Manual UNPUSHED diagnostic:** injected failures of either `git fetch` or `git ls-remote` print the new “no answer” warning and prevent the subsequent pipeline from running (`fleet/README.md:180–184`). This was not a re-review of the unchanged pipeline.
- **Bash and writer instructions:** `bash -n bin/fleet-audit` passes. The archive block's instruction to enter bash is appropriate. Adding a file after archive creation really does leave `tar -d` returning zero, supporting the new stop-writers explanation. These tests used quiescent LFS stores; neither two find passes nor tar comparison provides a filesystem snapshot under concurrent changes. Following links can enlarge a traversal; the existing host timeout/incomplete handling remains a conservative failure path, not evidence of a clean scan.

## Prioritized bottom line

1. **Fix B1 first:** the new external-store exception can satisfy both documented success markers without backing up the LFS contents. **Fleet-audit plus this README is not yet trustworthy for deletion.**
2. **Fix B2 before merge:** reserve remote archive directories atomically so a concurrent transfer cannot overwrite a backup already relied upon.
3. Correct N1 when convenient. **Not ready to merge into main.**
