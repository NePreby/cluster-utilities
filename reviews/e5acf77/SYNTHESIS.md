# SYNTHESIS: round 7, `fleet` @ e5acf77 (fixes since 55ebe14)

Panel: `review_claude-fable-5-1.md` (Fable 5.1) and `review_gpt-6.md` (codex / GPT-6).

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| fleet-audit (code) trustworthy? | **Yes**: no false `ok` in any link layout | Scanner FIXED for every static input tried | **Yes, the code is done** |
| README trustworthy for deletion? | No: G1 | No: G1, G3 | **No:** G1–G3, all in README procedure text |
| merge into main? | After G1 | After G1, G3 | **After G1–G3** |

F1 (symlinks inside the LFS store) is FIXED for both judges: all-linked, mixed, linked-object and
dangling-only layouts all end as `LFS-EXTERNAL` or `ERR`, never `ok`. F2 is FIXED for the reproduced
case (a pre-existing remote dir is refused and left untouched, also when `<other-host>` is this machine).

## Blocking (all README)

| # | Finding | Who | Fix |
|---|---|---|---|
| G1 | **"Archive the store itself" archives only links** when the store is linked (two of the four LFS-EXTERNAL causes). `tar` keeps symlinks, `tar -d` and the hash pass, and the README then says the objects "have their own verified copy". It's the exact round-6 mechanism, one paragraph down | **both** (reproduced) | drop that alternative: **LFS-EXTERNAL = keep the copy** (simplest, can't be followed wrong) |
| G2 | **`COPY OK` prints when the local hash is empty** (copy block run after a failed archive step: no SHA256SUMS → `h=""`; the remote sha256sum of a missing file → `""`; `"" = ""`) | Fable N1 (rated non-blocking; adjudicated blocking: it's a false "verified backup") | require `h` to be 64 hex chars before anything else |
| G3 | **Test-then-copy isn't atomic:** a concurrent transfer to the same name can slip between `test ! -e` and `rsync` | GPT-6 B2 (contrived) | reserve atomically with `ssh host "mkdir archives/$b"` (no `-p`: fails if it exists), then rsync into it |

## Non-blocking (cheap, folded in)

| # | Finding | Who | Plan |
|---|---|---|---|
| N1 | the comment says a dangling link makes find fail; it actually yields LFS-EXTERNAL | GPT-6 | fix the comment |
| N2 | `find -L` through a link out of the store can wander a huge tree into the host timeout | Fable | `-maxdepth 4` (LFS layout is `aa/bb/<oid>`) |
| N3 | `LFS:n` overcounts when a link points back inside the store | Fable | count is informational; note only |
| N4 | a partial rsync leaves a remote dir, and the chain then refuses forever | Fable | README: if the copy failed partway, remove that remote dir and rerun |
| N5–N7 | store location wording, the exit-2 nit, remote-shell claim wording | Fable | N5 moot after G1; reword the rest |

## Action plan
1. README G1–G3 + N4 wording; scanner N1 comment + N2 depth cap.
2. Round 8 on that diff. Merge if both judges report no Blocking findings.
