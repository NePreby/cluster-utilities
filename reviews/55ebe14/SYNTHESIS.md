# SYNTHESIS: round 6, `fleet` @ 55ebe14 (fixes since c3eeb4d)

Panel: `review_claude-fable-5-1.md` (Fable 5.1) and `review_gpt-6.md` (codex / GPT-6).

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| fleet-audit + README trustworthy for deletion? | No: 1 blocker (F1) | No: 2 blockers (F1, F2) | **No:** F1, and F2 because the fix is cheap |
| merge into main? | after F1 | after F1, F2 | **after F1 + F2** |

E1 holds for every layout round 5 named: absolute / `~` / relative-outside `lfs.storage`, a symlinked
`objects` dir, a linked worktree. E2 (unique archive dirs, tar -d, checksum) holds for Fable, and GPT-6
found a hole only at the destination (F2).

## Blocking

| # | Finding | Who | Class | Fix |
|---|---|---|---|---|
| F1 | **Symlinks below `objects/` are invisible.** `find -H` follows only the start path. A shard dir or object file that is a symlink isn't counted, and the containment check only resolves `objects` itself. All objects behind links → no LFS flag (**false clean**). Mixed → `LFS:n` without `LFS-EXTERNAL`, so the README allows the tar route, and tar stores only the link (**verified but empty archive**). Both judges reproduced it end to end | both | false clean / data loss via README | count with `find -L` (dangling or looping links → find error → ERR); **any** symlink inside the store → `LFS-EXTERNAL`; run the containment check whenever the store has files or links, not only when n > 0 |
| F2 | **The check on the destination can approve a different archive.** The `mktemp` name is unique only on the source machine. If that dir already exists on the other machine, `rsync --ignore-existing` keeps the old archive and the old SHA256SUMS, and `sha256sum -c` passes | GPT-6 (contrived: needs a pre-existing dir with the same random name) | data loss via README | refuse if the dir exists on the destination; compare the destination's computed hash with the **local** hash, not with the copied manifest |

## Non-blocking (cheap, folded in)

| # | Finding | Who | Plan |
|---|---|---|---|
| N1 | the archive block has no "bash syntax" note (the owner uses fish) | Fable | add it |
| N2 | `tar -d` can't see files added after `tar -c` | Fable | "stop writers first" already covers it; say why |
| N3 | separate git dirs (`--separate-git-dir`) show a spurious `WORKTREES:1 ERR` | Fable | pre-existing, safe direction. Note only |
| N4 | the manual UNPUSHED command prints nothing both when the server is unreachable and when everything is pushed | Fable | print a message on failure |
| N5 | the driver's own `tmp=$(mktemp -d)` is unchecked | Fable | check it |
| N6 | `LFS-EXTERNAL` has no documented way out | Fable | README: archive the external store separately with the same recipe, or keep the copy |

## Action plan
1. Fix F1 (+ D2-style fixtures: all-linked store, mixed store, external store holding only links), F2, N1, N2, N4–N6.
2. Round 7 on that diff. Merge if both judges report no Blocking findings.
