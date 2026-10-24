# REVIEW_REQUEST: round 2 code review of the `fleet` branch (commit 7a25a69) in cluster-utilities

**Task: adversarial CODE REVIEW of fixes.** A previous panel found serious bugs in this tooling. The
author says they're fixed. Your job is to (1) try to BREAK each claimed fix with concrete inputs, and
(2) find anything still wrong or newly introduced. Do not propose new features.

## Context
cluster-utilities holds the owner's notes and scripts for reaching compute (Alliance HPC clusters, see
AGENT.md) and now their own small fleet of machines over Tailscale: a laptop (`zephyrus-m16`) and a lab
workstation (`lkj008-psy`, ssh alias `lab`). Three more lab workstations (CachyOS already installed) are
about to be onboarded. All hosts run CachyOS (Arch-based). The owner's login shell is **fish** on every
host. No YAML parser (yq / PyYAML) is installed on the hosts, so the YAML files are read with awk.
Password ssh login is intentionally LEFT ENABLED on the lab boxes (owner's decision; do not flag it).

Goal of the branch: "one project, one host". Research repos live on lab machines, desktop/personal ones
on the laptop, infra repos on every host. Repos are moving from `~/code` (legacy) to `~/Projects`.

## Files to review (read them in full)
- `bin/fleet-audit`: run from the laptop. Its output is how the owner decides which copies are SAFE TO
  DELETE, so a false "clean" (unique work reported as ok, or a host/entry silently not checked) is the
  most serious class of bug.
- `bin/fleet-onboard`: run on a NEW machine at its keyboard, then self-elevates with sudo. It must be
  idempotent and must never cut the machine off the network or lock the owner out.
- `fleet/hosts.yaml`, `fleet/projects.yaml`: the awk-parsed data files.
- `fleet/README.md`: the procedure, including the deletion rule and the data-migration recipe.
- For house style: `bin/harden-sshd-keyonly`, `bin/cluster-status`.

## Claimed fixes, to verify adversarially
fleet-audit:
1. UNPUSHED counts commits reachable from local branches + tags + HEAD that no remote-tracking ref
   contains; a DETACHED flag is added.
2. Submodules (recursive) and linked worktrees are audited, with their counts added in.
   SUBMODULES:n / WORKTREES:n.
3. Untracked files are listed explicitly (`-unormal`), regardless of `status.showUntrackedFiles`.
4. IGNORED:n counts gitignored entries minus a list of regenerable names. `--size` adds their du.
5. `--fetch` runs `fetch --all --prune` (BatchMode ssh); a failure gives FETCHFAIL. LOCALREMOTE flags
   path or host:path remotes, using the configured URL.
6. Every git call is checked and gives ERR instead of zeros. Uses `--no-optional-locks`.
7. Dot-dirs, loose files (LOOSE:n), empty dirs, missing roots, and repos nested in non-git dirs are listed.
8. A per-host `#DONE` sentinel. An unreachable or partial host prints NOT AUDITED / INCOMPLETE on
   stdout, skips the not-found check, and exits 1. Unknown or inactive host ids exit 2.
9. Both YAML files are normalized per line (inline comments, trailing spaces, case). Unparsed lines and
   duplicates are reported. `[*]` means every host.
fleet-onboard:
10. resolv.conf switch: only if DNS works before, the stub exists and the backup succeeded. DNS is
    verified after; on failure the backup is restored and it exits 1. A rerun re-checks DNS.
11. The auth key is read hidden (`--authkey`) into a 0600 temp file and passed as `--auth-key=file:`.
12. The menu is installed as the user (`mkdir -p`/`tee`), so an existing 0700 ~/.config keeps its
    mode, and a symlinked destination is left alone.
13. `pacman -Syu`; tailscale handling decided on BackendState; refuses over SSH without `--force` and
    refuses a root login; sudo re-exec goes through bash with an absolute path; `--help` works without sudo.
README: the deletion rule matches the flags; a checksum-verified staging recipe replaces
`rsync --ignore-existing` + size compare.

## Also look for
- New bugs these changes introduced: `set -e`/`pipefail` interactions, `IFS=$'\t' read` collapsing
  empty fields, quoting, the `if br=$(…); then :; else [ $? -eq 1 ]` pattern, the awk regexes, and
  `find -prune` for nested repos.
- Anything that makes the deletion rule in README unsafe even when the tool behaves as written.
- Remaining false-clean paths not covered above (e.g. git notes/replace refs, reflog-only commits,
  sparse checkout, `assume-unchanged`/`skip-worktree` files, LFS objects, partial clones).
  Judge whether each actually matters for this owner's repos, and say so.

## Deliverable
Write `reviews/7a25a69/review_<model-slug>.md` containing:
- A table: claimed fix # → HOLDS / PARTIAL / BROKEN, with the input you tried.
- Findings split into TWO separate sections, each ranked most severe first:
  - `## Blocking`: must be fixed before this branch is merged or used, because it can cause data loss,
    lock the owner out or cut a machine off the network, or make fleet-audit report a false "clean"
    under the README deletion rule. For each one, say which of these it is.
  - `## Non-blocking`: worth fixing, but safe to merge and use without it (robustness, clarity, edge
    cases that can't cause the above, style). For each one, say in a line why it doesn't block.
  If a finding is borderline, put it under Blocking and say why. Each finding gives the file:line, what
  goes wrong, a CONCRETE trigger, and a suggested fix. Within each section, tag severity: BLOCKER
  (Blocking only) · MAJOR · MINOR · NIT.
- What you checked and found correct.
- Bottom line: (a) is fleet-onboard safe to run on the 3 machines? (b) can fleet-audit's output now be
  trusted for deletion decisions under the README rule? If not, what must change first?

Your value is INDEPENDENT judgment: do not read anything else under reviews/.
