# SYNTHESIS: round 5, `fleet` @ c3eeb4d (fixes since 7594a87)

Panel: `review_claude-fable-5-1.md` (Fable 5.1) and `review_gpt-6.md` (codex / GPT-6).

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| fleet-audit trustworthy for deletion? | **Yes**, no Blocking | No: 2 README blockers | **Not yet**, but only the README archive recipe is affected |
| merge into main? | **Yes** | After B1, B2 | **After E1, E2.** Small README fixes plus one flag |

All round-4 code fixes hold for both judges: D1 (unreadable, dangling and symlinked LFS stores), D3
(ENOENT suffix: tried ENOTDIR, ELOOP, a spoofed path, a path ending in the suffix), N1 (empty server →
UNPUSHED), N2 (cat-file failures → ERR), N3 (the manual command matches the tool), N4 (replace -d), N7.
Fable found **no false `ok` in any changed code path**. GPT-6's blockers are both in the README's new
archive-based clearance for LFS.

## Blocking (GPT-6; Fable didn't test these interactions)

| # | Finding | Class | Fix |
|---|---|---|---|
| E1 | **The tar recipe can't preserve an LFS store outside the repo dir**: an absolute `lfs.storage`, or a symlinked `objects` dir. tar stores the symlink, not the payload, and `tar -d` still says VERIFIED. Reproduced end to end, ending in loss (the store lived in a sibling checkout whose row was allowed) | data loss by following the README | scanner: a separate stop flag `LFS-EXTERNAL` when the resolved store isn't inside the repo's own git dir. README: the tar route clears LFS only for an internal store; LFS-EXTERNAL means keep the copy until that store has its own verified backup |
| E2 | **Archive name collisions:** `~/archives/<repo>-<date>.tar.gz` is the same for `~/code/x` and `~/Projects/x` on the same day; the second overwrites the first, also off-machine | data loss by following the README | a fresh unique dir per archive (`mktemp -d …/<host>-<repo>-<date>-XXXX`), a sha256 file in it, copy that dir off-machine without overwriting, and verify the checksum at the destination before deleting |

Adjudication: both are real and reproduced, and both happen by following the README to the letter,
which is the same standard as rounds 1–4. Fable's "no blocking" covers the code. These are about the
procedure.

## Non-blocking

| # | Finding | Who | Plan |
|---|---|---|---|
| N1 | the per-host timeout doesn't bound wall-clock: bash defers the TERM trap until the inner `timeout 90 git fetch` returns, and on ssh hosts the remote scan keeps running (its temp dir lingers until then) | both | qualify the comment. The host is already NOT AUDITED/INCOMPLETE, so it's conservative. A real fix is optional |
| N2 | README tar recipe: say "stop anything writing first" | Fable | fold in |
| N3 | README manual command: `set -o pipefail`, or a dead remote lists all history as unpushed | Fable | fold in |
| N4 | `lfs.storage` set to an empty string; `mktemp -d` failure | Fable | one-liners, fold in |

## Action plan
1. One small patch: E1 (flag + README), E2 (recipe), N2–N4, and the N1 comment.
2. Round 6 on that diff only. Merge if both judges report no Blocking findings.
