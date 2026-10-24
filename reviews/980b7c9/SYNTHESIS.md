# SYNTHESIS: round 3, `fleet` @ 980b7c9 (fixes since 7a25a69)

Panel: `review_claude-fable-5-1.md` (claude CLI, Fable 5.1) and `review_gpt-6.md` (codex / GPT-6).
grok was left out because it is still not signed in. Both judges reproduced their findings on fixtures;
GPT-6 also used real Git LFS 3.8 and a controlled DNS responder.

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| (a) fleet-onboard safe for the 3 machines? | **Yes** | **Yes** (keyboard procedure) | **Yes.** No blocking findings. Go one machine at a time with the reboot check |
| (b) fleet-audit trustworthy for deletion? | No: 2 blockers | No: 7 blockers | **No.** Fix the Blocking list below |
| (c) merge into main? | After B-list + N1 + docs | No | **Not yet.** One more fix round, scoped to the audit |

Round-2 items both judges call FIXED: A2 (row collapse), A4 (forge allow-list, every spoofing form
tried), A9 (REGEN), the `status -z` renames, and all onboard items (O1 SSH guard incl. a real
`sshd-session` ancestor, O-r1..4, O5–O7). PARTIAL: A1, A3, A5–A8, detailed below.

## Blocking

All are fleet-audit or README items. None are in fleet-onboard.

| # | Finding | Who | Class | Fix |
|---|---|---|---|---|
| C1 | **hosts.yaml still drops hosts with exit 0:** a host id at column 0, a tab-indented stanza, or any stray top-level line ends the `hosts:` section, and `!inh { next }` silently discards everything after it. This is the likely mistake when uncommenting the `# lab2:` template | both (Fable B2, GPT B6) | host not audited, but "all audited" | reject tabs; after `hosts:` any top-level line is an error (exit 2) |
| C2 | **An unreadable dir inside a repo hides its untracked files:** `git status` warns on stderr and exits 0, so the row is clean. Remote stderr is only shown when a host fails | Fable B1 | false clean | any `git status` stderr → ERR; print a host's non-empty scanner stderr even when it's `audited` |
| C3 | **An inaccessible parent dir makes a root look missing:** `-e` fails on EACCES → `ROOT-MISSING`, `audited`, exit 0 | GPT B7 | not audited, but "complete" | ROOT-MISSING only when the parent is readable and says ENOENT; otherwise `#ROOTERR` |
| C4 | **Stale tracking refs outside the fetch refspec still count as pushed:** `origin/old` survives a narrowed refspec, and a changed URL keeps old refs | GPT B1 | false clean | with `--fetch`, subtract the SHAs the server **advertises** (`git ls-remote`), not whatever is under `refs/remotes/<r>/`. ls-remote failure → FETCHFAIL |
| C5 | **Skip-worktree files whose name needs quoting** (tab, newline, quote, backslash) are tested under their quoted name, so the edit is missed | GPT B2 | false clean | `ls-files -v -z`; explicit `return 0` (also fixes N1) |
| C6 | **LFS:** a custom `lfs.storage` dir isn't looked at, and an unreadable object dir counts as 0 | GPT B3 | false clean | resolve `lfs.storage` (relative to the git dir); a find failure → ERR |
| C7 | **Blob replacement refs:** `git replace <blob> <newblob>` keeps unique content that commit counting can't see | GPT B4 | false clean (rare) | any `refs/replace/*` → `REPLACE:n` stop flag |
| C8 | **README says `git bundle --all` is a "full copy". It isn't:** reflog-only commits aren't in a bundle (reproduced), and annex payloads aren't either | GPT B5 | data loss by following the docs | replace with "archive the whole repo dir (`tar`, incl. `.git`), then verify the archive" |

## Non-blocking

| # | Finding | Who | Why not blocking | Plan |
|---|---|---|---|---|
| N1 | spurious ERR when the last index entry is skip-worktree and absent (sparse checkout) | both | false positive | fold into C5 |
| N2 | `LFS:n` can't be cleared by re-running: the local objects stay after a successful `git lfs push --all` | GPT N2 | the rule keeps the copy, so it's safe | README: LFS is cleared by a recorded `git lfs push --all <forge>` exit 0, not by re-running |
| N3 | `FLEET_AUDIT_FORGES` with a trailing space matches an empty host, so a local path counts as a backup. Both env vars are only read on the local host | Fable N2 | needs a malformed, hand-set env var | require a non-empty host; say in the docs that it's local only. Fold in |
| N4 | stale docs: the header and README still say `fetch --all`; the README's manual UNPUSHED check uses all remotes; it says "parser reports on stderr" | Fable N4 | docs, but the README is the rule | fold in |
| N5 | `pushInsteadOf` isn't checked (only matters without `--fetch`) | Fable N5 | `--fetch` resets the ref | also require the push URL to be on a forge. Fold in |
| N6 | onboard: the `--authkey` key is lost on the "reboot, then rerun" path | Fable N6 | usability | check tun before prompting for the key. Fold in |
| N7 | onboard: `/sys/module/tun` is absent with a built-in TUN (`=y`) | Fable N7 | linux-cachyos is `=m` (verified) | also accept `/dev/net/tun` when `/proc/config.gz` says `CONFIG_TUN=y`. Cheap |
| N8 | cosmetics: long non-GitHub URLs, a space in a SUBMODULE-NOBACKUP path splits the flag, `ignored-size:0` | Fable N8 | display | optional |

## Adjudication notes
- GPT's C4/C6/C7 are rare setups (narrowed refspecs, custom LFS storage, blob replacements). They are
  still blocking, because each is a reproduced false clean and each fix is small. That is the same
  standard as rounds 1–2.
- C8 matters beyond rarity: following the README literally loses data. It is a docs fix.
- Both judges independently cleared fleet-onboard. The 3-machine rollout does not need to wait for
  the audit fixes.

## Action plan
1. One audit patch: C1–C7 + N1, N3, N5 (code), C8 + N2 + N4 (README).
2. Optional onboard polish: N6, N7.
3. Round 4: the audit diff only. If both judges come back with no Blocking findings, merge `fleet`
   into main.
