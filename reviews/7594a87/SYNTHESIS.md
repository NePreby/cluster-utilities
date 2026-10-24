# SYNTHESIS: round 4, `fleet` @ 7594a87 (fixes since 980b7c9)

Panel: `review_claude-fable-5-1.md` (Fable 5.1) and `review_gpt-6.md` (codex / GPT-6). Both reproduced
their findings, with real git-lfs 3.8 among other tools.

## Verdict

| Question | Fable | GPT-6 | Adjudicated |
|---|---|---|---|
| (a) fleet-onboard safe for the 3 machines? | Yes | Yes | **Yes**, for the fourth round running. No onboard finding is blocking |
| (b) fleet-audit trustworthy for deletion? | Almost: 1 blocker | No: 3 blockers | **Not yet:** D1–D3 |
| (c) merge into main? | After D1 | After D1–D3 | **After D1–D3** |

Round-3 items both judges call FIXED: C2 (unreadable dir in a repo), C5 (quoted skip-worktree names, no
sparse ERR), C7 (REPLACE), C8 (tar instead of bundle: GPT restored a reflog-only commit, a private tag
and an LFS payload from it), N3/N5, onboard N6/N7. C1 and C4 hold for every host-dropping and stale-ref
case. Only diagnostics remain there (N-list below).

## Blocking

| # | Finding | Who | Class | Fix |
|---|---|---|---|---|
| D1 | **The LFS store precheck still counts an unreachable store as 0.** `[ -e "$st/objects" ] \|\| return 0` is also false on EACCES (`.git/lfs` mode 000, or an unreadable custom store dir) and on a dangling symlink. A symlinked `objects` dir isn't descended into (find doesn't follow a symlinked start path). Row `ok`, exit 0 | both | false clean | treat only a definite ENOENT as "no store", like roots; otherwise ERR. `find -H` (follow the start-point symlink) |
| D2 | **README: an exit-0 `git lfs push --all` doesn't prove the LFS content is on the forge.** `lfs.url` can point elsewhere (reproduced with a local endpoint), and `push --all` skips objects no ref references (e.g. staged then reset). Both were reproduced as "cleared", then lost | GPT B2 (Fable N5 related) | data loss by following the docs | LFS can't be cleared by a push: keep the copy, or archive the whole repo dir (tar, verified) before deleting |
| D3 | **The ENOENT test matches the path too:** `stat`'s message contains the path, so a path containing "No such file or directory" with EACCES gives ROOT-MISSING and exit 0 | GPT B3 | host not audited, but "complete" (contrived) | match only the message suffix `*": No such file or directory"`. Same helper for D1 |

## Non-blocking (folded into the same patch, all cheap)

| # | Finding | Who | Why not blocking | Fix |
|---|---|---|---|---|
| N1 | **An empty server repo** (created on GitHub, never pushed) gives `^missing` → ERR plus a repo-less `fatal:` line, instead of UNPUSHED:n | both | ERR is a stop flag | drop blank advertisement lines; only accept well-formed cat-file output |
| N2 | the cat-file pipeline's status and stderr are ignored (corrupt objects) | GPT N2 | the count only gets bigger | check the status; stderr → ERR |
| N3 | **The README's "see the actual commits" command uses cached refs**, so it contradicts the audit (prints nothing where the audit says UNPUSHED:1) | both | the tool still wins under the rule | use the same ls-remote negation as the tool |
| N4 | README `git replace -d <ref>` fails; it needs the replaced `<sha>` | Fable N3 | the flag stays until fixed | `git replace -l` / `git replace -d <sha>` |
| N5 | README tar recipe: run from `<parent>`, write the archive outside the fleet roots (or LOOSE flags it), verify against disk with `tar -df`, not just `-t` | Fable N4 | failures are loud | reword |
| N6 | `sed "s#^#$d: #"` breaks on `#`/`&` in a path; no trap for scanner temp files; the "scanner warnings" heading also catches ssh noise | Fable N8/N9 | cosmetic / robustness | awk prefix; EXIT trap; neutral heading |
| N7 | a trailing tab after `hosts:` is accepted (it's normalized away); the README says "any tab" | GPT N4 | can't drop a host | reword: tabs in indentation/content |
| N8 | a global `lfs.storage` makes every repo report the shared store's count | Fable N6 | noisy, not unsafe | note only |
| N9 | onboard: a mistyped key is only noticed after the upgrade; an absent TUN loops on "reboot, rerun" | Fable N7 | convenience | note only |

## Action plan
1. One audit patch: D1–D3 + N1–N7.
2. Round 5 on that diff. If both judges report no Blocking findings, merge `fleet` → main.
