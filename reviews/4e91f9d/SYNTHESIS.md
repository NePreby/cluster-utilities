# SYNTHESIS: review of `fleet` @ 4e91f9d

Panel (agent-utilities `run_reviewers.sh`, independent snapshots):
- `review_gpt-6.md` (codex / GPT-6)
- `review_opus-5-5.md` (claude / Opus 5.5)
- grok produced no output because its CLI is not signed in on the laptop (`grok login --device-code`). This
  round is a 2-judge panel.

Both judges reproduced their main findings on scratch repos and mocks. They agree on almost every
fault; they disagree only on severity labels. Nothing here was found by just one judge and left
unverified, except where marked.

## Verdict

| Component | Verdict |
|---|---|
| `fleet-audit` as a **"safe to delete" oracle** | **Not safe.** Several states report clean when they hold unique work (both judges, reproduced). |
| `fleet-audit` as an **inventory / placement report** | Usable now, if you read it as "where things are", not "what's safe to delete". |
| `fleet-onboard` | **Fix 4 small things first**, then fine for the 3 machines. GPT says "block rollout", Opus says "OK with (a)+(b)". The fix list is the same either way. |
| Docs (`lab-access.md`, AGENT.md, menu embed) | Correct. The fish menu embed was verified byte-identical by both judges. |

## Consensus findings: fleet-audit (fix before using it to decide deletions)

Both judges found each of these and reproduced it unless noted. Severity is the higher of the two labels.

| # | Problem | Line | Fix |
|---|---|---|---|
| A1 | **Detached-HEAD and tag-only commits are invisible.** UNPUSHED uses `--branches` only, and DETACHED is not a flag. | `fleet-audit:75` | `git log --branches --tags HEAD --not --remotes`, plus a DETACHED flag when HEAD isn't reachable from a remote |
| A2 | **Unpushed commits inside submodules are invisible.** | `:73-75` | recurse with `git submodule foreach --recursive`, or flag SUBMODULE for a manual check |
| A3 | **Linked worktrees:** their uncommitted work is not checked (Opus). Deleting the main checkout destroys the other worktrees' git dir (GPT). | `:73-74` | loop over `git worktree list --porcelain`; add a `WORKTREES:n` flag |
| A4 | **Stale remote-tracking refs count as "pushed".** There's no `--prune`, and fetch errors are silenced, so a branch deleted on GitHub still looks pushed. | `:71,75` | `fetch --all --prune`, a FETCHFAIL flag; only `--fetch` runs can clear a copy for deletion |
| A5 | **Git errors count as 0.** A corrupt index gives a clean row. | `:73-77` | check each command's exit status; add an ERR flag |
| A6 | **`status.showUntrackedFiles=no` hides untracked files.** | `:74` | pass `--untracked-files=normal` (or `all`) explicitly |
| A7 | **A failed or partial host scan exits 0,** and prints bogus "registered … but not found" lines. A typo'd host id prints an empty, clean-looking report. | `:96,105-108,128-136` | per-host `#DONE` sentinel; print `NOT AUDITED` on stdout; exit 1; reject unknown ids |
| A8 | **hosts.yaml parsing:** an inline comment on a host-id line (`lab2: # new`) merges that host into the previous one. Trailing whitespace after `active` silently drops the host. This is the exact edit made when uncommenting the lab2 template. | `:41-47` | strip comments and whitespace on every line before matching; warn on skipped hosts |
| A9 | **Dot-dirs and loose files in a root are never listed.** An empty root prints a bogus `*` row. This matters before `rm -rf ~/code`. | `:62` | `nullglob`/`dotglob`, and a per-root `LOOSE:n` row |
| A10 | **Ignored data gives no signal**, but that's where the research-data risk is (epilepsy: 67 GB). | `:73-74` | an `IGNORED:n` flag (`git status --ignored`), with `du` under `--size` |

**The README deletion recipe is wrong** (GPT #6, adopted). `rsync --ignore-existing` followed by
"compare sizes" can keep a different older file at the destination and still pass the check. Replace
it with an rsync into a separate migration directory (no `--ignore-existing`), checked with
`rsync -rcn --itemize-changes` (checksums), then resolve any conflicts by hand.

## Consensus findings: fleet-onboard (fix before the 3 machines)

| # | Problem | Line | Fix |
|---|---|---|---|
| B1 | **resolv.conf switch has no precondition, no DNS check and no rollback.** It breaks DNS if `DNSStubListener=no`, NM is unmanaged, or a hand-written campus resolv.conf is in use. A failure partway leaves the symlink in place, and a rerun then skips the restarts. Backup failure is ignored. | `:89-103` | require `[ -e "$stub" ]` and a successful backup. After the restarts run `getent hosts archlinux.org`; on failure, restore the backup and exit 1. Print the backup path |
| B2 | **The auth key is visible in argv** (`/proc/*/cmdline`) and the `TS_AUTHKEY=… ./fleet-onboard` form puts it in fish history. Worse with a reusable key. | `:112`, README | read the key with `read -rs`, or take a key file, then use `--auth-key=file:<0600 tmp>` and delete it on exit |
| B3 | **`install -d` changes an existing `~/.config` from 0700 to 0755** (both reproduced). | `:83` | create only the missing directories (`[ -d x ] \|\| install -d -m 700 …`), or `sudo -u "$user" mkdir -p` |
| B4 | **`pacman -S` without `-y`** gives 404s on a box with a stale sync DB, or a partial upgrade of openssh (Opus only, by reasoning; standard Arch guidance). | `:37` | `pacman -Syu --needed --noconfirm …` |

Lower priority (Opus only, by reasoning): a rerun right after the tailscaled restart can hit a bare
`tailscale up` "non-default flags" error, so branch on `BackendState` instead. Running the script over
SSH cuts its own session when tailscaled/NM restart, so warn if `SSH_CONNECTION` is set. `exec sudo
"$0"` fails if the script lost its exec bit, so use `bash "$(readlink -f "$0")"`. `printf > "$dest"`
as root follows a symlinked dest.

## Where the judges differ

- **Severity of A4/A6/B1.** GPT calls them BLOCKER; Opus calls them MAJOR/MINOR. **Adjudicated: treat
  them as blockers for deletion use.** Each one is a way for the tool to report a copy as safe when it
  holds unique work, which is the failure that matters most here.
- **Onboard rollout.** GPT: "block until DNS is validated and reboot-tested". Opus: "OK with B1+B2".
  **Adjudicated:** apply B1–B4, run it on ONE machine, reboot that machine and confirm `ssh lab2` +
  MagicDNS, then do the other two.
- **Registry gap (Opus #20, accepted):** the infra repos are listed as `[laptop, lab]`, so they will
  show MISPLACED on lab2+. Support `[*]` (all hosts) or document "append the new host id" in README
  step 4.

## Action plan, in order

1. **fleet-onboard:** fix B1–B4. These are small, and the 3 machines are waiting on them.
2. **fleet-audit, safety:** fix A1–A7 and A9, and change the README deletion recipe. Until this lands,
   no copy is deleted on the strength of fleet-audit output. Check by hand instead, with
   `git fetch --prune`, `git log --branches --tags HEAD --not --remotes`, `git status --ignored`,
   `git worktree list`, `git submodule status` and `ls -A`.
3. **fleet-audit, robustness:** fix A8 and A10, support `[*]`, then the NITs (`--help` range, truncated names,
   `--no-optional-locks`, `GIT_SSH_COMMAND='ssh -o BatchMode=yes'` for `--fetch`).
4. Re-run this panel on the fixed commit (`reviews/<new-commit>/`), with grok signed in to get a third judge.
