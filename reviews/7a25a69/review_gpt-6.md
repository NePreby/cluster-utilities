# Independent review of the fleet fixes

Reviewer: **Codex, based on GPT-6**. No more specific model version identifier is available to me.

**Recommendation: do not use this audit to authorize deletion.** Several ordinary-looking rows still pass the README's deletion rule while containing unique work. Onboarding's DNS rollback is substantially improved, but its SSH refusal can disappear across sudo.

I read the requested `REVIEW_REQUEST.md`, all of `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/hosts.yaml`, `fleet/projects.yaml`, `fleet/README.md`, both house-style scripts, and `AGENT.md`. I did not read any other review. Tests used disposable repositories outside this repository, Git 2.56.0, actual Bash/fish, and local bare repositories. HTTPS fixture URLs were redirected to those bare repositories using repository-local `insteadOf` settings; no fleet machines or external Git servers were contacted. Onboarding tests used a temporary script copy with its resolver paths redirected to temporary files and system/service commands mocked. These establish control-flow behavior, not actual CachyOS network health. No real package installation or service restart was performed.

## Claimed-fix matrix

“HOLDS” means the stated mechanism survived the specified tests; it does not certify the broader deletion guarantee.

| Fix | Verdict | Concrete input and observed result |
|---|---|---|
| 1. Branches + tags + HEAD; DETACHED | HOLDS | Made an unpushed commit on an inactive branch, a commit reachable only through a tag, and a detached HEAD commit. Each produced `UNPUSHED:1`; detached HEAD also produced `DETACHED`. The definition still trusts stale remote refs and excludes other local state: B1/B4. |
| 2. Recursive submodules/worktrees | PARTIAL | Nested initialized submodules produced `SUBMODULES:2` and added dirty/untracked/unpushed counts. An external linked worktree's untracked file was counted. After committing that file on its detached HEAD, the parent showed only `WORKTREES:1`, without `UNPUSHED`. The stop flags prevent deletion despite incomplete counts. |
| 3. Explicit untracked listing | HOLDS | Set `status.showUntrackedFiles=no`, then created an untracked file: `UNTRACKED:1`. |
| 4. Ignored entries and sizes | PARTIAL | Ignored `data/` was counted, `.venv/` excluded. An ignored `.ipynb_checkpoints/` containing unique work disappeared entirely (B6). An ignored `my data/` containing 1 MiB was counted but omitted from `ignored-size` (N3). |
| 5. Fetch and LOCALREMOTE | PARTIAL | Injected fetch failure produced `FETCHFAIL`. `/path`, `file:///path`, and `host:path` produced `LOCALREMOTE`; `user@localhost:path` and `ssh://user@localhost/path` did not. `skipFetchAll` and an orphan remote-tracking ref survived a successful fetch (B1/B3). |
| 6. Every Git failure becomes ERR | PARTIAL | Injected status and symbolic-ref exit 128 produced `ERR`. Injected `rev-parse -q --verify HEAD` exit 128 did not. Inspected `g()` and direct Git invocations for `--no-optional-locks`; they include it. |
| 7. All root entries/nested repos | PARTIAL | Dot-repo, loose file, empty directory, missing root, and `wrapper/branch` were listed correctly. A real unreadable root containing a file yielded zero entries and `audited`, exit 0 (B8). |
| 8. Host completion sentinel | PARTIAL | Mock SSH exit 255, a clean row without `#DONE`, and a timeout all produced the appropriate stdout failure and exit 1; no not-found checks ran for those hosts. Unknown/planned explicit IDs returned 2. Internal enumeration failures still produce `#DONE`; malformed active entries can vanish before launch (B7/B8). |
| 9. Flat YAML normalization/diagnostics | PARTIAL | Inline comments, trailing whitespace, `status: ACTIVE`, and `[*]` worked. Ordinary duplicate hosts returned 2; duplicate/unparsed project lines warned. Missing `ssh` collapsed TSV columns; a duplicate whose first status was empty was accepted. Malformed host indentation warned but still exited 0 (B7). |
| 10. Guarded resolver switch | HOLDS | Mocked failing pre-change DNS, absent stub, failed backup, post-switch DNS failure, and broken DNS on rerun. Preconditions preserved the original resolver; post-switch failure restored its contents and exited 1; rerun detected failure. See N5 for a masked service error. |
| 11. Hidden/file-based auth key | HOLDS | Used a controlling pseudoterminal with a fake key. Input was not echoed; mocked `tailscale up` received only `--auth-key=file:<path>`; file mode was 0600, contents matched, and the file was removed at exit. |
| 12. User-owned menu installation | HOLDS | Installation preserved an existing 0700 `.config`; rerun recognized identical content; a symlink destination and its target were left alone. Inspected user-context `mkdir`, backup, and `tee`; embedded menu passes `fish -n`. |
| 13. Upgrade/state/elevation/guards | PARTIAL | Mock log confirmed `pacman -Syu --needed --noconfirm`; root login refused; help worked before elevation; re-exec uses absolute path and Bash. A sudo mock removing `SSH_CONNECTION` reached NetworkManager restart without `--force` (B9). First failing `tailscale status --json` aborted instead of retrying (N1). |

## Blocking

### B1 — BLOCKER: successful fetch does not make every `--remotes` ref valid backup evidence

**Location:** `bin/fleet-audit:128–132`, `bin/fleet-audit:153–155`; deletion rule at `fleet/README.md:19–26`.

**Consequence: false clean and data loss.** The count subtracts every ref under `refs/remotes/`, including refs that the current fetch did not update or prune.

Two reproduced inputs, starting with a clean clone and a new local commit:

```bash
# A stale tracking ref not governed by any configured remote:
git update-ref refs/remotes/abandoned/main HEAD
# origin/main remains one commit behind HEAD.
```

Or push the new commit, rewind the bare remote's branch to its parent, and configure:

```bash
git config remote.origin.skipFetchAll true
```

In both cases `fleet-audit --fetch --all` returned 0, printed `audited`, and reported the repository `ok`. In the first case `git log origin/main..HEAD` still displayed the unique commit. In the second, fetching all remotes intentionally skipped the only configured remote. Custom/narrowed refspecs can leave the same class of stale evidence.

**Fix:** establish the comparison refs from successful, explicit fetches of the accepted backup remotes into a controlled namespace, or validate and restrict the existing refs to what this fetch actually refreshed. Reject skipped/narrow/inconsistent configurations when their coverage cannot be established. Do not equate a successful `fetch --all` exit with freshness of arbitrary remote-tracking refs.

### B2 — BLOCKER: index flags hide modified tracked files

**Location:** `bin/fleet-audit:113–125`.

**Consequence: false clean and data loss.** In otherwise pushed repositories I ran each of:

```bash
git update-index --assume-unchanged tracked
# In a second repository:
git update-index --skip-worktree tracked
```

Then I overwrote `tracked` with unique local contents. Both repositories produced `ok`, exit 0, after `--fetch --all`. Normal porcelain status is not a byte-level comparison when these index bits are set.

**Fix:** detect these bits without modifying the index and block deletion pending inspection, or independently compare materialized files with their indexed contents. Do not clear the bits in the user's checkout merely to perform the audit. Sparse checkout itself does not imply lost work; a materialized, edited skip-worktree file is the concrete failure here. This is relevant to configuration and research repositories where local configuration edits are commonly hidden from status.

### B3 — BLOCKER: adding a username or SSH URL scheme defeats LOCALREMOTE

**Location:** `bin/fleet-audit:145–151`.

**Consequence: false clean and data loss.** Set a repository's remote to itself through SSH:

```bash
git remote set-url origin user@localhost:/absolute/path/to/this/repo
```

Fetching creates `origin/main` from that same repository. There is no second copy. A fake SSH transport executing the real local `git-upload-pack` demonstrated a successful fetch and an info-only row. The same happened with `ssh://user@localhost/absolute/path/to/this/repo`. In contrast, the identical path without a username (`localhost:/...`) was correctly flagged.

The unconditional exceptions for `*@*:*` and almost all `*://*` URLs classify transport spelling as storage independence. `user@lab:path` is also entirely normal fleet syntax. Inspecting only the configured URL similarly cannot prove independence when an `insteadOf` rewrite changes the destination.

**Fix:** identify the destination host across URL forms and compare it against localhost, fleet aliases/names/IPs, and effective SSH destinations; inspect effective URL rewriting too. For deletion certification, an explicit set of accepted backup destinations is safer than accepting every username-bearing SSH URL. Unknown destination independence must block the row.

### B4 — MAJOR: unique Git state outside branch/tag/HEAD commit reachability is lost

**Location:** `bin/fleet-audit:128–132`, `bin/fleet-audit:166`; `fleet/README.md:22–26`.

**Consequence: false clean and data loss.** Independently reproduced:

* `git notes add -m 'only local experiment notes'`: `ok`; ordinary fetch did not upload the notes.
* Make a unique commit and `git reset --hard HEAD~1`: `ok`; the commit remained recoverable only through the local reflog.
* `git tag -am 'only local release metadata' v1` on an already pushed commit: `ok`; the annotation/tag object was never published.
* Create a replacement commit, reset to the original, `git replace <original> <replacement>`, and reset the worktree to HEAD: info-only output, despite the locally substituted content existing only in replacement/reflog state.

These are not failures of fix 1's narrow mathematical definition; they are failures of the README's broader assertion that an allowed row means the copy is dispensable. Reflogs are particularly relevant during branch cleanup and migration. There is no evidence that these particular owner repositories use notes or replace refs, so those cases are conditional, but the audit can detect their presence rather than silently granting deletion.

**Fix:** inspect nonstandard refs and tag objects, flag unpublished state, and surface locally recoverable commits that are not proven backed up. If intentionally discarded history is to be excluded, require an explicit reviewed discard decision or preserve a complete repository backup first. Do not claim a commit-count check covers all Git state.

### B5 — MAJOR: Git LFS payloads are not covered by the deletion certificate

**Location:** `bin/fleet-audit:153–166`; `fleet/README.md:19–26`; relevant scope in `AGENT.md` A.3 and B.7–B.8.

**Consequence: false clean and data loss.** With actual Git LFS, I tracked `*.bin`, committed a result file, and pushed Git refs with `core.hooksPath=/dev/null`. That bypassed LFS upload. There was one local LFS object and no remote LFS object directory. The audit successfully fetched Git refs, returned 0, and showed only `UNLISTED`, which the README explicitly permits for deletion. The worktree file matched its LFS pointer through the clean filter, so status did not protect it.

The guide explicitly discusses LFS and scientific data. I did not inspect the owner's project repositories and cannot assert current usage. This is a conditional blocker: existing LFS/annex repositories must not receive the ordinary deletion guarantee. Git-annex has the analogous separation between Git metadata and payload availability; I did not run an annex reproduction.

**Fix:** detect these storage mechanisms and block deletion unless payload availability elsewhere is verified by the appropriate tool or an independent verified copy. Merely fetching Git objects or comparing pointer commits is insufficient. Partial clone and sparse checkout alone are not demonstrated data-loss bugs: absent data already on a valid remote need not exist locally, and ordinary failed Git operations generally produce `ERR`.

### B6 — MAJOR: a directory name is not proof its ignored contents are regenerable

**Location:** `bin/fleet-audit:108`, `bin/fleet-audit:120–121`; `fleet/README.md:25–26`.

**Consequence: false clean and data loss.** A fully pushed repository ignoring `.ipynb_checkpoints/`, with a unique notebook checkpoint inside it, reported `ok`. It never entered the README's ignored-data migration path. Notebook checkpoints can preserve edits absent from the current notebook, especially after a revert or failed editing session. The blanket `env`, `venv`, and `.cache` exclusions have the same unproven assumption; the regex applies at any directory depth.

This matters for the stated research/data migration, including the ignored data and preregistration caches explicitly described in `fleet/projects.yaml:31–32`. I am not claiming all caches or all virtual environments need archival. I am saying the script cannot certify their contents disposable from a basename alone.

**Fix:** do not silently exclude potentially unique checkpoints/data from a deletion-oriented audit. Surface exclusions for explicit review, and document an affirmative disposal decision for regenerable artifacts before deletion. If exclusions remain, narrow them to verified project-specific cases and make the deletion rule conditional on reviewing them.

### B7 — MAJOR: malformed active hosts can disappear from a successful audit

**Location:** `bin/fleet-audit:60–87`.

**Consequence: a host is not audited, but the command reports successful fleet completion.** Add this stanza beside a valid laptop:

```yaml
  lab2:
    roots: ~/Projects
    status: active
```

The awk output contains an empty SSH field. Bash's whitespace-IFS `read` collapses the adjacent tabs: `ssh` receives `~/Projects`, `roots` receives `active`, and `st` becomes empty. The required-field check is never reached. Observed stderr: `(skipping host 'lab2': status <none>)`; stdout reports only the laptop as audited; exit status is 0.

A three-space host indentation also warned and disappeared with exit 0. A duplicate host whose first stanza lacked a status escaped the duplicate check because `known[$id]` was empty. Read/awk errors are not checked either. The shipped inventory is valid; the risk is especially relevant while adding the three new stanzas.

**Fix:** validate required keys and allowed statuses in awk before emitting records; fail with exit 2 on malformed inventory. Preserve empty fields or avoid whitespace-delimited interchange. Check key existence for duplicates, not a nonempty stored status. Propagate parser/read failures, reject zero eligible hosts, and do not turn unknown/missing statuses into intentional inactivity. This is a blocker under the request's explicit requirement that omitted hosts invalidate the audit, even if a careful reader might spot the stderr warning.

### B8 — MAJOR: the completion sentinel also certifies unreadable roots

**Location:** `bin/fleet-audit:212–215`, `bin/fleet-audit:224–234`, `bin/fleet-audit:280–285`.

**Consequence: entries are silently unaudited and the host is falsely declared complete.** As an unprivileged user, I created a root containing a unique loose file, changed the root mode to 000, and put its absolute path in the inventory. Output was `== locked: audited (0 entries)`, exit 0, with no `LOOSE` or error row. The not-found check then ran as though traversal had succeeded.

With `nullglob`, a failed directory expansion is indistinguishable from an empty root here. Similarly, `ls -A` errors are discarded and interpreted as `EMPTY`; `find` errors are discarded inside process substitution. `#DONE` establishes that execution reached the end, not that enumeration succeeded.

**Fix:** check directory enumeration/traversal success, report affected paths, and make such errors invalidate host completion. Preserve and inspect `find` status rather than hiding it in process substitution. A missing root may remain an explicit missing-root result, but inaccessible existing data must not look empty. This is borderline under the row-present deletion rule, which does protect an individually missing repository; I classify it as blocking because the advertised whole-host/root completeness and no-loose-files checks are false.

### B9 — MAJOR: the SSH refusal occurs after sudo may erase its evidence

**Location:** `bin/fleet-onboard:32–35`, `bin/fleet-onboard:53–57`, `bin/fleet-onboard:139–141`.

**Consequence: network/session loss without the required `--force` opt-in.** `SSH_CONNECTION` is checked only in the elevated process. With a normal sudo policy that does not preserve this variable, an SSH invocation passes that check. Default sudo environment reset constructs a restricted environment; arbitrary variables require preservation rules. See the upstream [sudoers command-environment documentation](https://github.com/sudo-project/sudo/blob/main/docs/sudoers.man.in).

I tested a sudo executable mock that removed `SSH_CONNECTION`, preserved the invoking-user identity, and re-executed Bash. Starting with `SSH_CONNECTION='1 2 3 4'` and no `--force`, the full mocked script returned 0 and reached `systemctl restart NetworkManager`. With the variable preserved, it correctly refused before package/service actions. Actual sudo policy on the new machines was not verified; a local noninteractive sudo probe required a password.

**Fix:** parse `--force` and reject SSH before self-elevation; retain a post-elevation check as defense in depth. Do not depend on sudo preserving the connection variable. This directly defeats a claimed network-safety guard, even though strict adherence to the keyboard-only README procedure avoids triggering it.

## Non-blocking

### N1 — MAJOR: the Tailscale retry loop aborts on its first failed status command

**Location:** `bin/fleet-onboard:184–187`.

**Trigger/result:** mocked `tailscale status --json` returned 1 immediately after the restart. The script exited 1 after one call, without sleeping, retrying, or running `tailscale up`. With `set -e` and `pipefail`, the unguarded assignment inherits the failed pipeline status.

**Fix:** explicitly capture a failed status attempt inside an `if`/guard, retry transient failure, and treat an unknown state after the deadline as a diagnostic failure rather than blindly attempting login.

**Why it does not block:** in the supported keyboard workflow, this fails visibly after a successful DNS check; it does not itself disable LAN networking, remove credentials, or issue a false deletion certificate. It should still be fixed before onboarding three machines to avoid predictable partial setup.

### N2 — MINOR: worktree/submodule accounting is narrower than advertised

**Location:** `bin/fleet-audit:168–185`.

**Trigger/result:** a unique commit on a linked worktree's detached HEAD produced `WORKTREES:1` but no `UNPUSHED`. Additional worktrees receive only `count_status`; their HEADs and their separately initialized submodules are not recursively inspected. Submodules do not get independent fetch/remote-policy checks, and their own linked worktrees are not traversed.

**Fix:** reuse a carefully bounded repository/worktree inspection routine or narrow the count descriptions. Keep the unconditional stop flags until all owned storage has been handled.

**Why it does not block:** `WORKTREES` and `SUBMODULES` are already prohibited by the README deletion rule, so these omissions do not turn those rows into allowed deletion candidates.

### N3 — MINOR: quoted ignored paths give incorrect sizes

**Location:** `bin/fleet-audit:115–121`, `bin/fleet-audit:199–201`.

**Trigger/result:** Git quoted the ignored path `my data/`. The scanner used the printed quotes as filename characters; `du` failed silently for that path. With a 1 MiB file there and another small ignored directory, the row said `IGNORED:2 ignored-size:4.0K`, despite total repo size exceeding 1 MiB.

**Fix:** use porcelain `-z` and NUL-safe parsing, including the special structure of rename records; propagate size-query failures rather than presenting partial totals as complete.

**Why it does not block:** `IGNORED:2` remains present and blocks deletion; the README no longer uses size equality as proof of successful migration.

### N4 — MINOR: optional Git probes still swallow unexpected errors

**Location:** `bin/fleet-audit:130`, `bin/fleet-audit:163–165`, `bin/fleet-audit:209–210`.

**Trigger/result:** making `rev-parse -q --verify HEAD` return 128 did not add `ERR`. The upstream probe also treats every failure as “no upstream.” The initial repository-identification probe's status is not checked. In contrast, the questioned `if br=$(...); then :; else ... $? ...` pattern worked: detached exit 1 and injected fatal exit 128 were distinguished correctly.

**Fix:** distinguish expected absence/unborn cases from operational errors and report the latter. Keep positive evidence for an empty/unborn repository rather than interpreting arbitrary failures as absence.

**Why it does not block:** the tested attached-branch case still counted branch refs; detached cases retain the `DETACHED` stop flag; lost upstream information affects `BEHIND`; failed repository identification normally falls back to `NOGIT`. I did not demonstrate an additional independently deletable row from these probe failures.

### N5 — MINOR: restart failure can be masked inside the DNS helper

**Location:** `bin/fleet-onboard:139–142`, `bin/fleet-onboard:167`.

**Trigger/result:** mocked `systemctl restart systemd-resolved` failed, while the subsequent NetworkManager operation and DNS lookup succeeded. The script printed `DNS still works` and exited 0. Because `restart_dns` runs as an `if` condition, errexit does not stop at its first failed command; the later command supplies the function's result.

**Fix:** explicitly return failure from a failed resolved restart (`... || return`) and preserve both service-operation results.

**Why it does not block:** a failing DNS lookup still enters the tested rollback path. The demonstrated masking requires the independent DNS check to succeed, so the test proves a misleading service outcome, not network loss.

### N6 — MINOR: make checksum-verification success explicit in the procedure

**Location:** `fleet/README.md:35–41`.

**Trigger:** the verification command can exit nonzero with errors on stderr; “prints nothing” is not a sufficient success condition. A rerun into an existing staging directory may also retain same-size/same-mtime wrong contents during the ordinary `rsync -aP` copy, although the checksum pass correctly detects them.

**Fix:** require exit 0, no errors, and no itemized differences; explain that differences must be resolved and verification repeated before deletion. Keep the source quiescent through the copy and verification.

**Why it does not block:** the new checksum comparison correctly detected equal-size different contents in my test and produced no differences after a verified repair. Errors are visible; the core recipe is a real improvement over size comparison.

## Checked and found correct

* Branch-only, tag-reachable, detached, dirty/untracked, stash, and ordinary ignored-data protections worked in the tested cases. In particular, `status.showUntrackedFiles=no` did not suppress `UNTRACKED`, and an actual stash produced `STASH:1`.
* Recursive initialized submodules were found. Dot-directories and a repository nested in a non-Git container were found; the container retained `NOGIT NESTED`, protecting loose sibling data. The bounded nested search can miss deeper repositories, but the containing `NOGIT` row still blocks deleting the container.
* Running the remote command through actual fish successfully handed the scan to Bash and retained the literal `~/Projects` display root. The ordinary inventory's roots do not exercise single quotes or whitespace inside path components.
* Transport timeout/failure and missing sentinel handling were fail-closed in the tested cases. A returned clean row did not override the final `INCOMPLETE` result.
* The shipped YAML files use the supported shape. `[*]` matched hosts without marking infra misplaced; status case normalization and inline comments worked. Invalid project rows/duplicates produced diagnostics rather than silently replacing the first entry.
* Auth-key handling, preserved menu-directory mode, symlink refusal, root-login refusal, help before sudo, Bash re-exec, and basic DNS rollback behaved as described in their tested scopes. Password authentication was deliberately left out of the findings, as requested.
* Both shell scripts pass `bash -n`; the menu passes `fish -n`. The scripts were not run against live fleet machines, and the three target machines' sudo, networking, LFS/annex, index flags, and Git-ref state remain unverified.

## Prioritized bottom line

1. **Deletion decisions: no.** The strongest counterexamples satisfy the actual README rule: exit 0, host `audited`, and `ok` or permitted info-only flags while unique data remains. First fix backup-ref provenance and remote identity (B1/B3), hidden working-tree edits (B2), and omitted Git/payload/ignored state (B4–B6). Fix inventory and traversal completeness (B7/B8) before treating fleet-wide success as meaningful. Until then, retain complete independent copies; an audit row is not a deletion certificate.
2. **Onboarding the three machines: not an unconditional approval of this branch.** The resolver safeguards passed the controlled failure tests, and the exact keyboard-only procedure avoids the SSH-guard defect. Fix the pre-sudo SSH check (B9) before general use, and fix the broken Tailscale retry (N1) before the three-machine rollout. Then proceed one machine at a time with the documented reboot and fresh SSH/MagicDNS checks; these mocked tests cannot certify actual network configuration.
3. **Keep the successful fixes.** The detached/tag/branch coverage, explicit untracked listing, stop flags for extra worktrees/submodules, transport sentinel, hidden key file, user-context menu installation, and checksum staging are useful improvements. They are not sufficient to support the present deletion guarantee.
