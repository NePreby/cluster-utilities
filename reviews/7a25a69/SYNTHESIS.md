# SYNTHESIS: round 2, `fleet` @ 7a25a69

Panel (agent-utilities `run_reviewers.sh`, independent snapshots):
- `review_claude-fable-5-1.md` (claude CLI, `AGENT_MODEL_CLAUDE=claude-fable-5-1`)
- `review_gpt-6.md` (codex / GPT-6)
- grok was left out because its CLI is still not signed in.

Both judges built adversarial fixtures and reproduced their findings, except where marked.

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| fleet-onboard safe for the 3 machines? | Yes at the keyboard. Fix the SSH guard + `modprobe tun` first | Not unconditionally. Fix the SSH guard + the tailscale retry first | **Fix O1 (blocking) and the O-rollout items, then go one machine at a time** |
| fleet-audit trustworthy for deletion? | Not yet, because of one bug (malformed host skipped, exit 0) | No, several false-clean paths remain | **No.** A3–A9 below must land first |

Round-1 fixes that both judges confirmed HOLD: detached/tag/branch UNPUSHED (#1), explicit untracked (#3), the
DNS switch guard/verify/restore (#10), the hidden auth-key file (#11), the user-owned menu install (#12),
the transport sentinel for ssh failure/timeout (#8 in part), and the checksum staging recipe.

## Blocking

"Blocking" means it can lose data, cut a machine off or lock the owner out, or make fleet-audit pass a
copy under the README deletion rule when the copy holds unique work.

| # | Finding | Who | Class | Fix |
|---|---|---|---|---|
| A1 | **A malformed active host is silently skipped and the run exits 0.** An empty awk field gives adjacent tabs, which `IFS=$'\t' read` collapses, so the fields shift and status becomes empty, which reads as "inactive". The trigger is a missing or misspelled `ssh:`/`roots:` or a 3-space indent, exactly while the 3 new stanzas are typed. A duplicate host whose first status is empty also escapes the duplicate check | both (Fable B1, GPT B7) | host not audited, but "all audited" | `-` placeholders for empty fields (or a `\x1f` delimiter); validate required keys and allowed statuses → exit 2; duplicate check on key existence; list skipped hosts on **stdout** |
| A2 | **The same tab collapse in the row parser** turns any flags into `ok` if a field is ever empty. Not triggerable with git 2.56 today | Fable N1 | latent false clean | same patch: `emit` substitutes `-` for empty args |
| A3 | **An unreadable root or dir looks empty.** A root with mode 000 gives `audited (0 entries)`, exit 0. A 000 dir gives `EMPTY`. `find`/`ls` errors are discarded | GPT B8, Fable N2 | entries not audited, but "complete" | check `-r`/`-x` and enumeration status; `ERR NOACCESS` row; an enumeration error invalidates host completion |
| A4 | **LOCALREMOTE misses `user@host:path` and `ssh://host/...`**, which is normal fleet syntax (and can even point at the repo itself) | both (Fable B3, GPT B3) | false clean | invert to an allow-list: only known forges (`github.com`, `gitlab.com`, `git.overleaf.com`, …) count as a real backup; anything else is LOCALREMOTE |
| A5 | **Refs that aren't real backup evidence still count as "pushed":** orphan `refs/remotes/<gone>/*` from a removed remote, `remote.X.skipFetchAll=true` (skipped by `fetch --all`) | GPT B1 (Fable N6d: rare) | false clean | compare only against refs of *configured* remotes; flag `skipFetchAll`/a narrowed refspec as FETCHFAIL-like |
| A6 | **Local-only refs outside branches/tags/HEAD:** `git notes`, `git replace`. Unpublished annotated-tag objects on pushed commits | GPT B4 (Fable N6: rare) | false clean (conditional) | count `--all` minus `refs/remotes/<configured>/*` minus `refs/stash` (Fable's command). That covers notes and replace. Tag objects on pushed commits stay non-blocking (N-list) |
| A7 | **`assume-unchanged` / `skip-worktree` files hide edits from status** | GPT B2 (Fable N6c: rare) | false clean (conditional) | cheap detection, `git ls-files -v` lowercase/`S` entries → `HIDDENIDX:n` flag (stop flag) |
| A8 | **Git LFS payloads:** pointer commits pushed, objects not (hook bypassed / push failed) | GPT B5 (Fable: LFS pushes via hook) | false clean (conditional) | detect LFS (`filter=lfs` in .gitattributes or `.git/lfs/objects` non-empty) → `LFS` stop flag; a deletion then needs `git lfs push --all` verified |
| A9 | **REGEN exclusions hide real data:** `.ipynb_checkpoints/` (unique notebook states), `.cache/`, bare `env/` | both (GPT B6, Fable N5) | false clean | drop `.ipynb_checkpoints`, `.cache`, `env` from REGEN; list the remaining names in the README |
| O1 | **The "refuse over SSH" guard can't fire.** It runs after `exec sudo`, whose `env_reset` drops `SSH_CONNECTION` | both (Fable B2, GPT B9; mock-verified by GPT) | claimed network-safety guard is broken | parse `--force` and check SSH **before** the sudo re-exec; keep the post-sudo check as well |

### Adjudication notes
- Fable rated A5–A7 "rare, skip" and GPT-6 rated them blockers. **Adjudicated: blocking, because each is a
  false clean and each fix costs a few lines.** Rarity is a reason for a cheap fix, not for none.
- A8 (LFS) is conditional: no LFS use is known in the owner's repos. Detection costs one check, so it is a
  stop flag rather than a full verification.

## Non-blocking

Safe to merge without these, but the **O-rollout** rows should be done before the 3-machine rollout,
because each makes the script exit 1 partway (recoverable with a rerun, no lockout).

| # | Finding | Who | Why not blocking | Fix |
|---|---|---|---|---|
| O-rollout-1 | `tailscale status --json` failing right after the restart aborts the script under `set -e`; the retry loop never retries | GPT N1 | fails visibly after DNS is verified | guard the call; retry; treat unknown-after-deadline as an error |
| O-rollout-2 | `-Syu` upgrades the kernel. `tun` is a module (`CONFIG_TUN=m`) that can't be loaded until reboot, so tailscaled can't start | Fable N9 (verified config) | rerun after reboot works | `modprobe tun` before the upgrade; if `/usr/lib/modules/$(uname -r)` is gone, say "reboot, then rerun" and exit |
| O-rollout-3 | **The DNS probe doesn't test resolv.conf at all:** `getent` goes through nss-resolve, so "DNS works" is true whatever the symlink says. Direct readers (tailscale itself, Go, containers) aren't checked | Fable N8 (verified nsswitch) | the default stub listener answers anyway | before switching, require the stub listener to be up (`ss -lun` shows 127.0.0.53:53, or `resolvectl query`), and probe via the stub after |
| O-rollout-4 | `NeedsMachineAuth`: `tailscale ip -4` fails → exits before the hand-off text | Fable N10 | rerun works | `|| true` with a placeholder |
| O5 | a failed `restart systemd-resolved` is masked by the later NM restart | GPT N5 | the DNS check still gates rollback | `|| return` in `restart_dns` |
| O6 | only the leaf is checked for a symlink; a symlinked `conf.d` parent writes into the dotfiles repo | Fable N11 | shows up as UNTRACKED there | check `readlink -f` of the parent dir |
| O7 | a relative `resolv.conf` symlink isn't recognized → a new backup every run | Fable N12 | harmless relink | `readlink -f` |
| N1 | a linked worktree's detached-HEAD commits aren't counted, and submodules' worktrees aren't traversed | GPT N2 | WORKTREES/SUBMODULES are already stop flags | reuse the repo inspection for each worktree HEAD |
| N2 | `ignored-size` drops paths that git quotes (spaces) | both | IGNORED:n still blocks; size isn't proof | `-c core.quotePath=false`, or `-z` parsing |
| N3 | `rev-parse --verify HEAD` / upstream probe failures aren't ERR | GPT N4 | no deletable row demonstrated | distinguish absence from error |
| N4 | projects.yaml: 4-space entries dropped silently; unknown host ids like `[Lab]` accepted | Fable N3 | only produces extra warnings | report them; validate ids against hosts.yaml |
| N5 | nested repos deeper than 4 levels are missed, and repo-in-nested-repo is double counted | Fable N7 | the parent stays `NOGIT NESTED` | raise the depth; de-dup prefixes |
| N6 | reflog-only commits, unpublished tag objects on pushed commits | GPT B4 (part) | reflog = discarded work by definition; tag objects are rare | document in README as out of scope |
| N7 | README wording: say what UNDECIDED/EMPTY/`ignored-size` mean for deletion; the REGEN list; the checksum pass needs exit 0 plus no output; `_incoming-*` will show UNTRACKED until merged | both | docs | edit README |
| N8 | `p` isn't `local`; `sort` splits names on spaces; a symlinked repo dir hides behind LOOSE | Fable N13 | nits | tidy up |

## Action plan

1. **One patch for fleet-audit completeness:** A1 + A2 + A3. Same root causes (whitespace-IFS interchange,
   unchecked enumeration).
2. **One patch for fleet-audit backup evidence:** A4 + A5 + A6 + A7 + A8 + A9, plus the README rule and
   the N7 wording.
3. **fleet-onboard:** O1 (blocking) + O-rollout-1..4 + O5–O7. Re-test with mocks, including a sudo
   mock that strips `SSH_CONNECTION`.
4. Optional: the remaining non-blocking items N1–N5, N8.
5. Round 3 only on whatever changed in 1–3, ideally with grok signed in.
