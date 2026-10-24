# Independent review of 7594a87

Model: **GPT-6 (Codex)**. A more specific model version/snapshot is not available to me.

**Recommendation: do not merge yet; do not use this version to authorize deletion.** The onboarding changes passed the scoped checks, but the audit and deletion instructions retain three blocking failure modes.

I read `bin/fleet-audit`, `bin/fleet-onboard`, and `fleet/README.md` in full, plus the actual inventories and relevant repository guidance. I read no other file under `reviews/`. The checkout has no usable Git metadata, so change scope comes from the request's inlined diff, checked against the real files.

Tests used disposable directories outside this repository, an unchanged copy of `fleet-audit`, generated inventories, real Git repositories, and local bare repositories reached through the supplied forge/SSH test hooks. No real fleet host or forge was changed. Tests ran unprivileged with Git 2.56.0 and Git LFS 3.8.0. Onboarding tests mocked administrative commands; a pseudo-terminal exercised the actual hidden-key read. I did not run a system upgrade or change networking.

## Round-3 fix verification

| Item | Verdict | Concrete input tried and result |
|---|---|---|
| C1: inventory silently drops hosts | **PARTIAL** | Column-zero host, tab-indented host, stray top-level key, content before `hosts:`, missing section, and duplicate section all exited 2 before scanning. `hosts:` followed by a trailing tab was accepted and exited 0: the literal “all tabs” claim is broader than the implementation. Host omission cases are fixed; see N4. |
| C2: unreadable directory inside repository | **FIXED** | Created `unreadable/unique`, then chmodded its directory to 000. Git's exit-zero warning produced `ERR`, and the warning appeared beneath the audited host. |
| C3: inaccessible root parent | **PARTIAL** | Ordinary chmod-000 parent produced `INCOMPLETE`, exit 1. A nonexistent root produced `ROOT-MISSING`; a dangling root symlink produced `INCOMPLETE`. A permission-denied pathname containing the ENOENT wording still produced `ROOT-MISSING`, exit 0; B3. |
| C4: stale tracking refs count as backups | **PARTIAL** | A local-only commit also referenced by stale `origin/old`, with the fetch refspec narrowed to `main`, correctly produced `UNPUSHED:1`. Annotated/peeled tags worked. 12,000 advertised pull refs worked. A failing `ls-remote` produced `FETCHFAIL`. Empty advertisement and corrupt-object/error handling still have diagnostic defects; N1–N2. |
| C5: quoted skip-worktree names; sparse checkout error | **FIXED** | Skip-worktree files containing a tab, quote, and newline, plus a dangling symlink, produced `HIDDENIDX:4`. Removing the four paths to simulate absent sparse entries produced `ok`, without a spurious `ERR`. |
| C6: custom/unreadable LFS storage | **PARTIAL** | `lfs.storage=custom` with one object under `.git/custom/objects` produced `LFS:1`; chmod 000 on `objects` produced `ERR`. Chmod 000 on `.git/custom` instead produced `ok`. A symlinked `objects` directory also produced `ok`; B1. Relative/absolute path interpretation was checked against `git lfs env`. |
| C7: replacement blobs | **FIXED** | Created a replacement blob with `git hash-object -w --stdin` and `git replace <old-blob> <new-blob>` in an otherwise backed-up repository: `REPLACE:1`. |
| C8: bundle called a full copy | **FIXED** | Archived and extracted an ordinary repository containing a reflog-only commit, a private annotated tag, and an in-directory LFS payload. Tar creation/listing both exited 0; the restored commit, tag object, and payload were present. This validates the tested layout, not external storage referenced by links/configuration. |
| N3: empty forge host | **FIXED** | Local-path remote plus a forge list containing doubled spaces produced `LOCALREMOTE`. |
| N5: push URLs | **FIXED** | One forge push URL plus `user@lab:copy` produced `LOCALREMOTE`. A `pushInsteadOf` rewrite to the lab also produced `LOCALREMOTE`. |
| N2: clearing IGNORED/LFS | **PARTIAL** | The IGNORED exception is explicit and points to checksum verification. A real exit-zero LFS upload to a local endpoint, and an unreferenced LFS object skipped by `push --all`, invalidate the LFS exception; B2. |
| N4: stale README wording | **PARTIAL** | The main explanation now describes advertisements correctly, but the new troubleshooting command still uses stale tracking refs. In the C4 fixture it exited 0 with no output while the audit reported `UNPUSHED:1`; N3. |
| Onboard N6: delayed key prompt | **FIXED** | With no usable TUN, the mocked original block stopped after upgrade/service enablement, before reading a key. With usable TUN, the pseudo-terminal showed upgrade and service startup preceding the hidden prompt. The input was not echoed and the EXIT trap removed its key file. |
| Onboard N7: built-in TUN | **FIXED** | Tested loaded-module state, absent module plus gzip configuration containing `CONFIG_TUN=y`, and `CONFIG_TUN=m` without a loaded module. The first two reached key input; the last exited with reboot guidance. A large compressed configuration also passed without a pipefail/SIGPIPE false failure. |

## Blocking

### B1 — BLOCKER: inaccessible LFS store parents still produce `ok`

**Class: false clean under the README rule; potential loss of unique LFS content.**

**Location:** `bin/fleet-audit:267` and `bin/fleet-audit:269`.

The existence precheck conflates “does not exist” with “cannot traverse a parent.” It returns success before the new checked `find` ever runs. This leaves the specific C6 safety promise incomplete.

**Reproduction:** Start with an otherwise clean repository whose commit is advertised by its forge. Create one local object at `.git/custom/objects/aa/data`, then:

```bash
git config lfs.storage custom
chmod 000 .git/custom
# Run fleet-audit --fetch --all as the ordinary owner.
```

Observed: the repository row was `ok`, the host was `audited (1 entries)`, exit 0, no warning. Restoring access made the same object produce `LFS:1`. Making only `objects` unreadable correctly produced `ERR`, demonstrating that the hole is the early precheck, not the later error handling.

There is a second enumeration hole: moving `.git/lfs/objects` to `.git/actual-objects` and replacing it with a symlink to that directory produced `ok` despite the retained payload. Default `find` does not descend into a command-line symlink. Both locations were inside the repository, so deleting it would delete the data too.

**Suggested fix:** distinguish confirmed absence from inaccessible paths before returning zero; all indeterminate existence/access checks must return an error. Resolve or explicitly follow the store's starting symlink and reject unresolved links. Preserve failure status when traversing the resolved store. Add fixtures for inaccessible ancestors and a symlinked object store, alongside the already-working unreadable-`objects` case.

### B2 — BLOCKER: exit-zero `git lfs push --all` does not justify the new blanket deletion exception

**Class: data loss when following the README literally.**

**Location:** `fleet/README.md:28` and `fleet/README.md:32`–35; interaction with `bin/fleet-audit:188`–200.

The new text explicitly clears `LFS` after one successful command. Neither its destination nor its coverage is established by that command's exit status.

**Reproduction A — destination:** I committed a real LFS-tracked `payload.bin`, copied its Git commit to the fixture forge without uploading the LFS object, then configured a separate local LFS endpoint:

```bash
git init --bare .git/local-lfs.git
git config lfs.url "file://$PWD/.git/local-lfs.git"
git lfs push --all origin
```

The upload exited **0**. The payload appeared under `.git/local-lfs.git/lfs/objects`; the forge had no LFS payload. The audit still showed only `LFS:1`, host audited, exit 0. The README now permits clearing that flag and deleting the copy, which deletes both local payload copies. The Git remote's fetch/push URLs remain forge URLs throughout, so N5's checks do not prevent this.

**Reproduction B — coverage:** Used `git lfs clean` to create another valid local object without adding its pointer to a ref, then ran `git lfs push --all origin`. It exited **0**, but the object remained only in the source store. The audit showed `LFS:2`, which the new instruction also permits clearing. An equivalent user workflow is staging an LFS file and then resetting it before committing; no reflog-only commit is required.

Git LFS documents separate LFS URL overrides and limits `push --all` to objects reachable from the selected local refs. These are distinct from uploading every file counted by this scanner. See the [Git LFS configuration manual](https://raw.githubusercontent.com/git-lfs/git-lfs/main/docs/man/git-lfs-config.adoc) and [push manual](https://raw.githubusercontent.com/git-lfs/git-lfs/main/docs/man/git-lfs-push.adoc).

**Suggested fix:** remove the unconditional exit-zero clearance. Require verifying the effective LFS upload destination is an independent approved backup, and accounting for every local object the user intends to preserve. Unreferenced objects need explicit object-ID upload with verification, a verified archive/copy, or an explicit decision to discard them. Document those limits before permitting deletion.

### B3 — MAJOR: the ENOENT test matches the pathname, not just the error

**Class: falsely completed host audit. Borderline because the pathname is unusual, but blocking under the request's explicit no-silently-unchecked-host criterion.**

**Location:** `bin/fleet-audit:389`–391.

GNU `stat` includes the path in its diagnostic. The substring match searches the entire diagnostic, so an EACCES error can match the ENOENT words contained in the filename.

**Reproduction:** Set the fixture's `HOME` to a directory named `/tmp/.../No such file or directory`, create its `Projects` child, chmod the home directory to 000, and use the valid inventory value `roots: ~/Projects`. Tilde expansion occurs inside the scanner, so whitespace splitting of the inventory does not invalidate this input.

The real error ends in `Permission denied`, but includes `No such file or directory` inside the quoted path. Observed: `(root) ROOT-MISSING`, `== test: audited (1 entries)`, exit **0**, instead of `INCOMPLETE`. Nothing under the inaccessible root was checked.

**Suggested fix:** inspect the actual error category. If keeping the GNU/English message approach, match the diagnostic's final error suffix, not an arbitrary substring that can occur in the path, and check the `stat` exit status. Keep permission errors and ambiguous results as `#ROOTERR`.

## Non-blocking

### N1 — MINOR: an empty advertisement creates a bogus revision

**Location:** `bin/fleet-audit:228`–231.

**Trigger:** A valid empty bare remote, with the ordinary wildcard fetch refspec and two local commits. Fetch and `ls-remote` succeed, but the advertisement is empty. The here-string supplies a blank record; `cat-file` emits ` missing`, and awk sees `missing` as field 1 rather than field 2. It writes `^missing` into the revision input.

Observed: `ERR` with `fatal: bad revision '^missing'`, instead of `UNPUSHED:2`.

**Fix:** filter empty advertisement records before `cat-file`, and accept only well-formed object/type results. An empty negative set should count all eligible local commits.

**Why it does not block:** `ERR` prevents deletion; this is a false alarm and missing diagnostic count, not a false clean.

### N2 — MINOR: advertisement filtering suppresses corruption diagnostics and ignores pipeline failure

**Location:** `bin/fleet-audit:228`–229.

**Triggers tested:** (1) An advertised blob under `refs/pull/42/head`, outside the fetch refspec, with its local loose object overwritten with invalid zlib bytes. Git's real `cat-file --batch-check` exited 0, emitted a `missing` record, and wrote inflate/unpack errors to stderr. Those errors disappeared; the audit reported `ok`. (2) A PATH shim making only `cat-file` exit 128. The pipeline failure was ignored and the audit reported `UNPUSHED:1`, without `ERR`.

The scanner has pipefail but not errexit. A failing pipeline needs an explicit status check; redirecting stderr also hides real corruption even when Git exits zero.

**Fix:** capture and check the pipeline's status and `cat-file` stderr; propagate errors to `ERR`. Distinguish an ordinary absent object from corruption diagnostics.

**Why it does not block:** the reproduced failure drops negative revisions, which conservatively increases the unpushed set. The corrupt blob in the first fixture already exists on the server. I did not demonstrate loss of unique work through this path; it is a reporting/error-contract defect.

### N3 — MINOR: the README's new diagnostic reproduces the stale-ref mistake

**Location:** `fleet/README.md:145`–152.

**Trigger:** The C4 narrowed-refspec fixture: local-only commit B, server containing only A, stale `refs/remotes/origin/old` pointing to B. The audit correctly reports `UNPUSHED:1`. Running the README's exact fetch/log sequence produces no log output because `--remotes=origin` still negates B through the stale ref.

**Fix:** make this diagnostic use the same server-advertised negative set as the audit, or clearly label it as a cached-ref approximation that can miss the reported commits.

**Why it does not block:** the audit's `UNPUSHED` stop flag remains; the misleading troubleshooting output does not itself clear it under the rule.

### N4 — NIT: the literal “any tab is exit 2” claim is not true

**Location:** `bin/fleet-audit:80`–84; `fleet/README.md:155`–158.

**Trigger:** A valid inventory beginning with `hosts:\t\n` rather than `hosts:\n`. Trailing-whitespace normalization removes the tab before the tab check; the audit succeeds.

**Fix:** either inspect tabs before normalization or describe the restriction as applying to structural/content tabs after harmless whitespace/comment normalization.

**Why it does not block:** this tested exception does not omit a host or change which roots are scanned.

## Checks that passed and limits

- `bash -n` passed for both scripts.
- The new `rev-list --stdin` negative revisions work with the command-line `--exclude ... --all` options in the tested stale-ref and annotated-tag cases. I found no reversal of positive/negative semantics.
- A 12,000-ref advertisement completed successfully; these refs shared a known commit SHA, so this tests advertisement size and deduplication, not 12,000 distinct object traversals.
- Failed server interrogation produced `FETCHFAIL`; unreadable worktree contents produced `ERR`; neither was silently cleared by an otherwise successful host scan.
- NUL-delimited hidden-index handling and replacement-blob flags passed the concrete fixtures above.
- Ordinary successful audit runs left no temporary files in a dedicated test `TMPDIR`. The onboarding EXIT trap removed the test key file. Abrupt process termination was not tested.
- The ordinary whole-directory tar restored content omitted by a bundle, including the reflog-only commit and local LFS payload.
- The changed onboarding block preserved upgrade → TUN check → daemon start → hidden-key prompt ordering. Its built-in-TUN branch worked under `set -euo pipefail`. I found no new owner-lockout or network-cutoff path in these changes. This is a scoped code/mock assessment, not validation on the three physical machines.

## Prioritized bottom line

1. **Fix B1 and B2 first:** both allow unique LFS data to be treated as safely backed up. Fix B3 before trusting the all-host completion guarantee.
2. **Fleet-onboard for the three machines:** yes, the round-4 changes are acceptable for keyboard-run onboarding, subject to the README's existing per-machine reboot and SSH verification. No blocking onboarding finding from this review.
3. **Fleet-audit trustworthy for deletion under the README rule:** **no**. The reproduced `ok`/successful-manual-clearance cases violate that rule's safety premise.
4. **Ready to merge into main:** **no** until B1–B3 are resolved. N1–N4 can follow separately.
