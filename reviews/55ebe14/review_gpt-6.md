# Independent review of 55ebe14

Model: **Codex, based on GPT-6**. No finer model version identifier is exposed to this session.

I read `bin/fleet-audit` and `fleet/README.md` in full. Scope is the supplied c3eeb4d..55ebe14 diff and its interactions with the deletion rule. This workspace has no Git metadata, so I could not independently verify the commit identities. I read no other file under `reviews/`.

Tests used disposable fixtures outside the repository, actual Git repositories, GNU tar, rsync, Bash, and fish. Full audit tests used an unchanged copy of `bin/fleet-audit`, a synthetic local-host inventory, and the documented forge/SSH test hooks. The SSH hook ran real `git-upload-pack` locally; no production host or forge was contacted. Destination transfer tests used local directories to reproduce rsync and checksum behavior; one also executed the destination command through fish. Only this review is a persistent repository change.

## Round-5 verification

| Item | Verdict | Concrete inputs tried and result |
|---|---|---|
| E1: prevent tar from falsely clearing externally stored LFS | **PARTIAL** | Absolute external `lfs.storage`, an external `objects` symlink, a linked worktree, and a separate external Git directory correctly set `lfsext=1`. However, an internal `objects` directory containing an external shard symlink produced `LFS:1` without `LFS-EXTERNAL` when one regular object was also present. With only symlinked shards, a full `--fetch --all` run printed `ok`, `audited`, and exited 0. See B1. |
| E2: prevent same-name archive collisions and verify the transferred archive | **PARTIAL** | Archiving two different `x` directories produced distinct local `mktemp` directories and successful local comparisons. With an older, self-consistent archive and manifest already at the selected destination basename, `rsync --ignore-existing` exited 0 and destination verification printed `x.tar.gz: OK` for the wrong archive. See B2. |

## Blocking

### B1 — BLOCKER: descendant LFS symlinks bypass the new archive-safety check

**Locations:** `bin/fleet-audit:303`, `bin/fleet-audit:307–310`; deletion authorization at `fleet/README.md:32–38` and `fleet/README.md:69–77`.

**Class:** false `clean` and an archive procedure that falsely establishes that LFS data is saved elsewhere.

`find -H` follows a symlink supplied as its starting path, but does not follow symlinks encountered below that path. Consequently, symlinked object files and shard directories contribute nothing to `n`. The new containment check only resolves `<store>/objects`, and only runs when `n > 0`. It does not establish that the individual objects are inside the archive boundary.

**Concrete trigger tested:** a clean repository with its only commit fetched from the test forge, plus an orphan LFS object containing `orphaned LFS bytes, never uploaded\n`. Its actual SHA-256 is:

```text
f86c030e26897568b9ca541550f402489ffb1e86ade2cb2bbbdae9b7704b1e2d
```

The filesystem layout was:

```text
Projects/x/.git/lfs/objects/f8 -> /outside/external-objects/f8
/outside/external-objects/f8/6c/f86c030e26897568b9ca541550f402489ffb1e86ade2cb2bbbdae9b7704b1e2d
```

With no regular objects alongside that link, the unmodified full audit reported:

```text
x ... ok
== test: audited (1 entries)
# process exit: 0
```

Adding one ordinary, correctly hashed LFS object under a different shard changed the row to **only `LFS:1`**, with no `LFS-EXTERNAL` or `ERR`. Following the archive recipe then succeeded:

```text
tar create: exit 0
tar compare: exit 0
x.tar.gz: OK
```

The archive contained the `f8` symlink and the ordinary object, but **no member containing the external object's bytes**. `tar -d` correctly verifies the link, so it cannot rescue this check. An external `lfs.storage` whose `objects` directory contains only symlinked object files also returned `rc=0 lfs=0 lfsext=0`: the new external-store check was skipped entirely.

Removing `x` alone does not unlink an external target. The demonstrated safety failure is that an unsaved LFS object is declared clean or treated as independently archived; the target remains dependent on the original machine. That directly violates the review's false-clean criterion and the README's backup rule.

**Scope:** the `find -H` counting behavior predates this diff. The in-scope failure is that the new `LFS-EXTERNAL` guard and the new README exception rely on that incomplete count and certify these layouts as archive-safe. This is an incomplete E1 fix, not a claim that the underlying traversal bug was newly introduced.

**Suggested fix:** validate the entire LFS object tree, including symlinked shards and leaves, before allowing the tar exception. Check an existing store's location independently of the regular-file count. Resolve descendant links and reject external targets; unresolved links, traversal errors, and cycles must fail closed. A conservative implementation can emit a stop flag for every unvalidated store symlink instead of trying to prove containment. Do not merely switch to `find -L` while continuing to validate only the top-level directory: that would count external objects without preventing their omission from tar. Add regression cases for mixed stores, all-link stores, and an external store containing only links.

### B2 — BLOCKER: destination verification can approve an entirely different archive

**Locations:** `fleet/README.md:67`, `fleet/README.md:71–77`.

**Class:** a README procedure that can authorize deletion without backing up the selected source.

`mktemp -d` reserves a name only on the source machine. It does not reserve that basename on the receiver. `rsync --ignore-existing` preserves both an existing archive and its existing `SHA256SUMS`. The subsequent check uses that destination manifest, so an old archive and its matching old checksum pass together. The local `VERIFIED` and remote `OK` can describe different byte sequences.

**Concrete trigger tested:**

1. Archive `code/x`, containing `old payload`, into one fresh directory. Archive `Projects-two/x`, containing `new unique payload`, into another fresh directory. Both local comparisons print `VERIFIED`.
2. Seed `other:archives/<second-directory-basename>/` with the first archive directory's contents, including its `SOURCE` and `SHA256SUMS`. This represents an already occupied destination name; no source-name collision is needed locally.
3. Run the documented transfer for the second directory and verify the receiver's manifest.

Observed results:

```text
rsync -a --ignore-existing: exit 0
sha256sum -c SHA256SUMS: x.tar.gz: OK, exit 0
destination x/data: old payload
SHA256(source archive) == SHA256(destination archive): false
```

There is no requirement to compare `SOURCE`, and it would be stale too. Both stated deletion conditions hold even though `new unique payload` was never copied. A naturally occurring random-name collision is unlikely; nevertheless this belongs under Blocking because the collision case silently produces exactly the false verification that E2 claims to prevent. The reproduction deliberately supplies the occupied destination rather than pretending to have observed a random collision.

**Suggested fix:** verify the receiver's archive against the **current source manifest**, for example by passing the local `"$d/SHA256SUMS"` on SSH stdin and running `sha256sum -c -` in the destination directory. Use appropriate path quoting and explicit shell handling. A mismatch must prevent deletion even when rsync itself returned 0. Alternatively, allocate an exclusive fresh directory on the receiver and abort if it cannot be reserved. Preserve the no-overwrite behavior; an occupied destination should require a new destination or fail verification. Test a pre-existing archive/manifest pair whose contents differ from the source.

## Non-blocking

None identified within the requested change scope. The following successful checks do not reduce either blocking finding.

## What I checked and found correct

- **Ordinary external stores:** an absolute external store and an external top-level `objects` symlink with regular objects both returned `lfs=1 lfsext=1`.
- **Gitfiles/worktrees:** a linked worktree using the main repository's populated store and a repository initialized with an external `--separate-git-dir` both set the external flag.
- **Submodules:** `.git/modules/sub/lfs/objects` was correctly internal when the archive boundary was the parent repository, and external when the boundary was the submodule directory itself.
- **Containment boundaries:** a sibling named `prefix-extra` was not mistaken for a child of repository `prefix`. A top-level `objects` symlink resolving to another actual directory inside the repository was accepted.
- **Repository symlinks:** `count_lfs` normalized an aliased repository path correctly. At the full scanner level, a symlink entry directly under a root produced `LOOSE:1` and no deletable repository row. I did not treat that entry as an archive-eligible repository.
- **Empty configuration:** `lfs.storage=` counted the default `.git/lfs/objects` store, rather than the normal Git object database.
- **Dangling top-level store link:** returned failure, which the caller maps to `ERR`.
- **Scanner temporary-directory failure:** injected failure only in the scanner's `mktemp -d`, leaving the outer reporting directory usable. The full command exited 1 and printed `NOT AUDITED (exit 1): scanner: mktemp -d failed`.
- **Normal archive path:** two equal repo basenames produced distinct directories; local tar comparisons passed. Running the archive block under Bash launched from fish, with the destination command evaluated by fish, produced `VERIFIED` and `x.tar.gz: OK` for a fresh destination. Thus B2 is not a shell-syntax objection.
- **Manual UNPUSHED command:** independently injected failure at `git fetch` and `git ls-remote`. In both cases the documented chain exited nonzero and produced no stdout; the downstream history pipeline did not run.
- **Stopping writers:** the new instruction makes the intended archive consistency precondition explicit. All archive reproductions used quiescent inputs, so neither blocking finding requires a concurrent writer.

## Prioritized bottom line

1. **Fix B1 first:** the audit can still print `ok` for unsaved LFS content, and the new no-`LFS-EXTERNAL` exception can approve an archive missing that content.
2. **Fix B2 before relying on the archive route:** remote `OK` must prove equality to the current source archive, not merely internal consistency of whatever already exists remotely.

**(a) Fleet-audit + README trustworthy for deletion? No. (b) Ready to merge into main? No: two blocking findings remain.**
