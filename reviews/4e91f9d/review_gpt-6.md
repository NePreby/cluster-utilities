Independent review by **GPT-6**. No more specific model version is exposed in this session.

**Recommendation: do not merge this branch for deletion decisions or onboard the three machines yet.** The audit can return zero unsaved-work counts for several forms of unique work, and onboarding can replace working DNS with an unusable resolver without recovery.

I read the complete contents of `bin/fleet-audit`, `bin/fleet-onboard`, `fleet/hosts.yaml`, `fleet/projects.yaml`, `fleet/README.md`, `bin/harden-sshd-keyonly`, `bin/cluster-status`, `AGENT.md`, and `lab-access.md`, plus this review's `REVIEW_REQUEST.md`. I did not read another file under `reviews/`. Repository implementation files were not changed. Reproductions used disposable local Git repositories, the extracted scanner, and controlled driver/service mocks; no privileged onboarding operations were executed. Test tools were Git 2.56.0, Bash 5.3.20, and fish 4.9.3.

Findings follow in severity order. For scanner reproductions, “zero counts” means DIRTY, UNTRACKED, UNPUSHED, BEHIND, and STASH are all zero unless stated otherwise. A registered, correctly placed row with these counts becomes `ok` under `--all` and disappears from the default report (`bin/fleet-audit:123–124`).

**1. [BLOCKER] Git failures become clean results. — `bin/fleet-audit:69–78`, especially `73–77`**

The scanner neither enables error checking nor checks the exit status of its Git queries. Errors are redirected away, and `wc -l` turns an unsuccessful command with no output into zero. The final `printf` succeeds, so the host scan also succeeds. Adding `pipefail` alone would not fix this: each failed query must be handled explicitly.

Concrete reproduction: create a clean checkout with a remote and replace its `.git/index` with the bytes `bad index`. Direct `git status --porcelain` fails with `fatal: .git/index: index file smaller than expected`. The actual scanner emits a normal `git` row with all five counts zero and exits 0, with no error on stderr. A corrupt/unreadable index must not certify the working tree; it can contain unique edits that Git was unable to inspect. Failed history and stash queries have the same problem. `rev-list HEAD..@{u} || echo 0` also conflates an absent upstream with other failures.

Suggested fix: capture and validate every required Git command's status before using its output. Emit an explicit inspection-failed state, never `ok`, for any unresolved query; propagate an incomplete-audit exit status. Handle “no upstream” as its own expected state rather than treating every revision-walk failure as zero.

**2. [BLOCKER] Branch-only history misses detached commits, tag-only commits, and detached linked-worktree history. — `bin/fleet-audit:72–75`**

`git log --branches --not --remotes` includes local branch tips, but not a detached HEAD or commits reachable only through local tags. Printing `DETACHED` in the branch column does not set a safety flag.

Concrete reproduction: in a fully pushed checkout, run `git switch --detach`, then `git commit --allow-empty -m ONLY-HERE`. The scanner reports `DETACHED` with zero counts. A second reproduction creates a commit while detached, tags it `unpublished`, and returns to the pushed branch: that also reports zero counts. Both lose unique history if the checkout is deleted.

Linked worktrees add a deletion dependency. I created a detached worktree outside the configured roots, committed there, and scanned only the main checkout. The main checkout reported zero counts; the other worktree's `.git` file pointed into `base/.git/worktrees/external`. Deleting `base` destroys the shared Git metadata/object store backing the other checkout. Merely recognizing `.git` files in worktrees that happen to be under a scanned root does not address this.

Suggested fix: include HEAD and relevant local refs, including tags, in the saved-history assessment. Inspect `git worktree list --porcelain` and each worktree's HEAD, and prohibit declaring a main checkout deletable while another worktree depends on its Git directory. Unsupported repository arrangements should produce an explicit unsafe/unknown result.

**3. [BLOCKER] A clean superproject can hide unpublished submodule history. — `bin/fleet-audit:62–78`**

Only the top-level repository is inspected. Superproject status describes the submodule's checked-out commit and worktree, not every branch or stash in the submodule's own repository.

Concrete reproduction: add and commit a submodule, push the superproject, create an unpublished branch/commit inside the submodule, then return the submodule to the commit recorded by the superproject. The scanner reports the parent with zero counts. Independently running `git rev-list --count --branches --not --remotes` inside the child returns **1**. Deleting the parent also deletes the submodule's object database in `.git/modules`, losing that branch. A submodule stash or ignored data presents additional omissions.

Suggested fix: audit initialized submodules recursively, including their local refs, stashes, and worktrees, and propagate unsafe/unknown status to the parent. At minimum, block deletion clearance for a checkout containing unaudited submodules.

**4. [BLOCKER] User Git configuration can hide untracked files. — `bin/fleet-audit:73–74`**

The untracked query uses `git status --porcelain` without overriding `status.showUntrackedFiles`. Porcelain format does not force untracked-file reporting.

Concrete reproduction: `git config status.showUntrackedFiles no`, then create an unignored `only-here.txt` in an otherwise clean, pushed checkout. The scanner returns `UNTRACKED=0`; the tracked-file query deliberately excludes untracked files too. This is an ordinary performance setting for large research repositories, not a malformed repository. Similarly, repository settings that ignore submodules can suppress dirty-submodule evidence.

Suggested fix: request untracked files explicitly, e.g. `--untracked-files=all`, and override submodule-ignore settings for the working-tree checks. Do not use the owner's presentation/performance preferences as the audit's safety policy. This complements, rather than replaces, the recursive-history fix above.

**5. [BLOCKER] Stale remote-tracking refs certify commits that are no longer saved remotely, even with `--fetch`. — `bin/fleet-audit:71,75`; `fleet/README.md:73–75`**

`--remotes` examines local cached refs. The fetch does not request pruning, and its failure or timeout is silently ignored. Consequently the documented “not on any remote” test can actually mean “was once referenced by a cached remote branch.”

Concrete reproduction: push a unique branch from checkout A, then delete that branch remotely from checkout B. Leave A's local branch and stale `origin/<branch>` intact. Running the actual scanner with fetching enabled gives `UNPUSHED=0`. In the fixture, an explicit `git fetch --prune` changes the same history count to **1**. The remote no longer advertises a recovery ref for that work; any server retention of unreachable objects is not a backup guarantee. A failed fetch or a remote force-push can likewise leave misleading cached state.

Suggested fix: require successful refresh/pruning of the remotes used to establish saved history before giving deletion clearance, and visibly mark failed or unverified remote checks. Keep a non-fetching report if desired, but label it as cached information that cannot certify deletion. Correct the README's deleted-branch claim: the current fetch does not necessarily remove that tracking ref.

**6. [BLOCKER] The ignored-data migration recipe can discard different contents at an existing path. — `fleet/README.md:19–21`; context `fleet/projects.yaml:30–31`**

The README correctly discloses that ignored data is not audited. Its prescribed substitute, `rsync -aP --ignore-existing` followed by comparing sizes, is insufficient for deletion. Existing destination files are skipped even when their contents differ, and equal size is not content equality. This matters directly to the documented research caches and 67 GB migration.

Concrete reproduction: source `results.bin` contains `NEW!`; destination `results.bin` contains `OLD!`. The documented rsync command exits 0, both files remain four bytes, and their contents remain different. Deleting the source after that size check loses its unique result. Aggregate directory sizes are even weaker.

Suggested fix: require content comparison of every retained source file and explicit preservation/resolution of conflicting versions before deleting a source. Copy into a separate migration directory or otherwise keep both versions; use checksums/content comparisons to verify recovery, with writers stopped or a consistent snapshot. Do not solve this by blindly removing `--ignore-existing` and overwriting the destination's possibly unique data. Audit `ok` should explicitly mean Git checks only, not complete backup verification.

**7. [BLOCKER] DNS replacement has neither a sufficient precondition nor rollback, and reruns can skip recovery. — `bin/fleet-onboard:89–103`**

An active `systemd-resolved` service does not establish that its stub listener works or that it has usable upstream DNS. The script replaces `/etc/resolv.conf` before restarting services, ignores backup failure, and has no rollback or DNS verification.

Concrete trigger: resolved is active with `DNSStubListener=no`, while the existing plain `/etc/resolv.conf` points at a working DNS server. Switching clients to the stub breaks their DNS. Disabling the listener while leaving the daemon active is explicitly supported by [systemd's resolved.conf documentation](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html#DNSStubListener=). Another supported arrangement obtains upstream DNS from a plain resolv.conf; resolved stops reading that file as an upstream source when it is changed to its own stub symlink, as documented in [systemd-resolved.service](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html).


Separately, I ran the exact DNS block with mocked commands and an injected failure of `systemctl restart systemd-resolved`: it backed up the file, replaced the link, then exited 1 without restoring anything. If an interrupted run leaves the symlink installed but NetworkManager integration unfinished, the next run takes the “already points at the resolved stub” branch and skips all three restarts. `is-active` also does not verify that resolved will be available after reboot.

Suggested fix: establish a working stub, upstream resolution, and persistent service configuration before switching clients; require a usable backup; perform the switch with recovery on failure; and verify ordinary DNS plus the intended tailnet resolution afterwards. Reruns must validate/recover the desired state rather than equating a symlink target with completed setup. The target path and restart sequence match [Tailscale's default NetworkManager/resolved advice](https://tailscale.com/docs/reference/linux-dns); the failure is applying that recipe without establishing its assumptions or recovering from partial execution.

**8. [MAJOR] Inventory comments on a host declaration silently associate the wrong machine with a host ID. — `bin/fleet-audit:41–47`**

Inline comments are stripped from selected fields, but not from host declarations before matching them. An unrecognized declaration leaves the previous `id` active while subsequent fields overwrite that host's connection and roots.

Concrete input:

```yaml
hosts:
  laptop:
    ssh: local
    roots: ~/Projects
    status: active
  lab: # workstation
    ssh: lab
    roots: ~/Projects
    status: active
```

The actual parser returns only `laptop<TAB>lab<TAB>~/Projects`. It scans the lab as “laptop,” does not scan the local laptop, and skips lab's registered-but-not-found check because no `lab.tsv` exists. This is worse than merely rejecting an unsupported YAML spelling. Trailing whitespace on `status: active` also prevents the exact `st=="active"` comparison from matching.

Suggested fix: normalize supported comments/whitespace before recognizing stanzas, and validate every declaration and required field. Reject an unsupported line with a diagnostic instead of silently reusing the previous host. The current checked-in inventory parses correctly; the failure is an ordinary annotated edit during onboarding.

**9. [MAJOR] Failed/incomplete host scans still exit successfully and report repositories as absent. — `bin/fleet-audit:96,105–108,128–136`**

The failure marker produces a stderr warning, but the driver's eventual exit status does not incorporate it. The missing-project check treats the existence of a TSV file as proof of a complete scan; redirection creates that file even when SSH never connects.

Concrete reproduction with a mocked SSH exit 255: stderr correctly says `scan failed`, stdout says `example: registered for lab but not found there`, and the process exits **0**. Absence was never established. A timeout after partial output can also leave normal rows and false absence messages from an incomplete host scan. Separately, `fleet-audit typo-host` exits 0 with only a header, silently scanning nothing.

Suggested fix: track completion independently from output-file existence, aggregate scan failures into a nonzero exit, and report missing projects only for successfully completed scans. Label incomplete host results as unknown. Validate requested host IDs before starting so a spelling error cannot masquerade as a successful empty audit.

**10. [MAJOR] Root enumeration omits hidden repositories and silently tolerates missing roots. — `bin/fleet-audit:60–62`**

The `"$r"/*/` glob excludes dot-prefixed directories. A repository such as `~/Projects/.experiment` with unpublished commits produces no row at all. If it is unregistered, the final placement check cannot recover the omission. Missing configured roots are silently skipped, so a typo or unavailable mount also looks like an empty scope rather than a scan failure. Conversely, an existing empty root yields a bogus `*` row with `NOGIT UNLISTED`, because `nullglob` is not enabled; I reproduced both the hidden omission and phantom row.

Suggested fix: enumerate real directories including hidden ones, with well-defined handling of empty roots, and distinguish an intentionally absent legacy root from an unavailable or invalid configured root. Do not make completeness claims for a root that was not inspected.

The separate `kirby/katam` case is conservative about deletion: it emits `kirby NOGIT`, blocking deletion under the README's rule, but never lists or audits `katam`. Thus `AGENT.md:27` and `fleet/README.md:11` overstate “every repo.” Either enumerate nested repositories or narrow that claim while preserving the NOGIT warning.

**11. [MAJOR] The auth key is expanded into a process argument. — `bin/fleet-onboard:111–112`; `fleet/README.md:25–27,37`**

Preserving `TS_AUTHKEY` across sudo is deliberate and useful, but expanding it into `tailscale up --auth-key=<secret>` exposes the literal credential in the new process's command line. On a normal Linux procfs without restrictive process visibility, another local user can observe it through process listings or `/proc/<pid>/cmdline`. Reusable keys, explicitly recommended for several machines, retain value after the first enrollment. The README's literal assignment can also persist the key in shell history.

Suggested fix: supply the key through a root-only temporary file or protected descriptor using Tailscale's supported `--auth-key=file:...` form, remove it on exit, and avoid teaching users to type the secret into persistent command history. The file form is defined in the [up command's primary source](https://raw.githubusercontent.com/tailscale/tailscale/main/cmd/tailscale/cli/up.go). Avoid inheriting the key into unrelated child processes once it is no longer needed. The script does not explicitly echo the key or enable shell tracing; this finding concerns argv/history exposure, not an observed secret log.

**12. [MAJOR] Installing the menu relaxes existing home-directory privacy permissions. — `bin/fleet-onboard:81–85`**

`install -d` without a mode applies its default `0755` to existing directories too. It is not equivalent to “create only if missing.”

Concrete reproduction: an existing `.config` is mode `0700`, with a `0644` file whose privacy relies on that parent directory. Run the script's `install -d -o ... -g ...` pattern for `.config`, `.config/fish`, and `.config/fish/conf.d`. The existing `.config` becomes **0755**, which I verified. With a traversable home directory, other local accounts can now read that file. This happens on the normal first installation or an update when the menu contents differ, even though the intended change is only a login menu.

Suggested fix: preserve ownership/modes of existing parent directories and create only missing directories with deliberate user-owned permissions. Install or replace the menu as the invoking user where possible. A nonsecret menu being `0644` is fine; weakening an existing private parent is not.

**13. [MINOR] The row format and display do not safely identify unusual repository names. — `bin/fleet-audit:67,78,108,117,124`; root quoting `90,94–95`**

Ordinary spaces inside repository basenames work: a real `repo with spaces` fixture retained its name and untracked count. Tabs/newlines in basenames are emitted verbatim into the TSV/line protocol, however, so they become field/record separators. The formatter also truncates basenames to 38 characters: two different directories sharing that prefix are indistinguishable in a deletion-oriented report. Root tokens containing a single quote break the hand-built remote quoting, even though whitespace-separated roots are the documented format.

Suggested fix: use an unambiguous internal representation and escape names for display, or reject unsupported control characters/root syntax explicitly. Always provide an untruncated identifier for the directory to which a safety result applies. Whitespace inside root paths is currently outside the documented space-separated schema; do not pretend adding shell-style quotes to that field makes it supported.

**Checks that passed, and limits of the review**

- Both scripts pass `bash -n`. The audit's `read -r -d '' SCAN` returns nonzero at EOF, but the audit deliberately lacks `set -e`, so it continues. Onboarding explicitly handles the same behavior with `|| true`; neither heredoc is an immediate errexit bug.
- I extracted the actual MENU assignment, evaluated it in Bash, and compared its result with the embedded text including the final newline: they matched byte-for-byte. `$c`, the nested quotes, and `case '*'` survive. `fish --no-config -n` accepts the result. The interactive/SSH/TMUX/HERDR_ENV/NO_TMUX guards are present, and unavailable herdr has a fallback. This verifies preservation and syntax, not an interactive end-to-end login.
- I invoked the remote command shape through fish with the scanner arguments. Fish passed literal `~/Projects` and `~/code` to Bash; the scanner's HOME substitution handles them there. Ordinary root tokens are correctly quoted for this path. It does not accidentally rely on fish performing Bash syntax or tilde expansion inside single quotes.
- A failing `readlink` on a plain file inside the onboarding `if [ "$(readlink ...)" != ... ]` condition does **not** abort under `set -e`; I tested this. An inactive NetworkManager on the left of `&&` is also not itself an errexit failure. A failed restart on the right is a real abort and is covered by finding 7.
- The checked-in YAML parses as intended: exactly laptop and lab are active; commented-out lab2 is ignored; field comments and project inline comments are removed; `epilepsy` becomes `lab,laptop`; infra multi-host lists and `[?]` are recognized. Membership comparison uses comma boundaries, avoiding accidental `lab`/`lab2` substring matches. Finding 8 concerns inputs that the parser silently misinterprets, not a claim that the existing files already misparse.
- `--branches` does include local branches without upstreams; an upstream is not required for the existing UNPUSHED calculation. Non-Git containers get NOGIT, and ordinary worktrees can be recognized via `rev-parse` rather than requiring `.git` to be a directory. These useful properties do not resolve the history/dependency omissions above.
- A wholly failed SSH invocation is visibly warned about on stderr, not explicitly printed as an `ok` host. The actual defects are the success exit status, misleading absence checks, and handling of partial output. Missing-project checks correctly avoid hosts deliberately excluded from the invocation.
- Self-elevation preserves the named auth environment variable, and the normal sudo path uses `SUDO_USER` to choose the invoking user's home and login shell. The resulting menu file is chowned to that user/group. Non-fish shells skip menu installation. The script intentionally leaves sshd authentication settings alone; password login is not a finding. Existing key-only configuration would also be left intact, so the README's “leaves password login on” assumes the starting machine permits it.
- The normal Tailscale logged-out path is placed inside `if`, so a nonzero status triggers `tailscale up` without an errexit abort. Upstream [status implementation](https://raw.githubusercontent.com/tailscale/tailscale/main/cmd/tailscale/cli/status.go) returns failure for logged-out/stopped states, but accepts Starting as well as Running; status success is not a complete connectivity test. I did not validate live enrollment, installed package versions, or actual host DNS/service state.
- `fleet-onboard:121` only retains two lines following the health heading and can print “no warnings” after a failed status command. Its final message should not be treated as a comprehensive health check. `--size` is disk-usage information, not a content-integrity check. The audit also inherits Git's limitations for ignored data and other deliberately hidden content; the report cannot establish an independent recoverable backup merely from an `ok` row.
- `bin/harden-sshd-keyonly` supplies a useful comparison: it validates before applying and has an explicit failure path, whereas the new DNS block switches first. I did not expand this review into a new audit of the existing helper or the unrelated Alliance procedures.

**Prioritized bottom line**

1. **Block deletion use first:** fix findings 1–6. Failed inspection, incomplete repository coverage, hidden untracked files, and stale remote state must never yield deletion clearance. Replace the ignored-data size-check recipe with verified preservation of contents and conflicting versions.
2. **Block unattended rollout to the three machines:** make the DNS change validated, recoverable, and rerunnable (finding 7), protect the auth key, and preserve existing directory permissions (11–12). Test failure/retry and reboot behavior on one machine before repeating the procedure.
3. **Before merging the fleet workflow:** make inventory parsing and scan completeness explicit, correct host-failure/absence reporting, and remove ambiguous directory identification (8–10, 13). Keep the successful fish/heredoc and ordinary-quoting behavior covered while making these changes.

**As submitted: not safe to merge and use for the stated deletion/onboarding workflow.** A default-path smoke test would miss the confirmed data-loss cases; passing the targeted failure cases above is the minimum basis for reconsideration.
