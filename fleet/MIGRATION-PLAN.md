# Repo migration plan (paused, 2026-10-24)

Goals: `~/code` → `~/Projects`, and **one project, one host** (research → lab; desktop/personal → laptop;
infra on every host). Owners are recorded in `fleet/projects.yaml`. Before deleting any copy, follow the
deletion rule in `fleet/README.md`: a `bin/fleet-audit --fetch --all` run that exits 0, with the row present
and no stop flags. IGNORED and LFS data need the manual checks the README describes.

Counts below are a snapshot. Re-run `bin/fleet-audit --fetch` before starting.

## Status
Nothing has been moved or deleted yet. The tooling (fleet-audit, projects.yaml) is done and on `main`.

## Steps, in order

### 1. Back up laptop-only work (data-loss risk; do this first)
- Git, but no remote: `acrobots`, `bot`, `football`, `sim-engine`. `youtube-audio-downloader` has no commits.
  → Create private GitHub repos under `yjmantilla` and push. Ask before each push.
- `kirby/katam`: 47 commits on no remote, plus 12 worktrees. Ask whether these commits should go on GitHub.
- About 25 non-git folders in `~/code`, roughly 70 GB (bonsai 14 GB, convert-to-text 13 GB,
  amica-python-zenodo, pdf-converter, study-permit, mitacs, red-teaming-aaai-paper, …). Decide each one:
  make it a repo, archive it (README recipe), or delete it.

### 2. Push unpushed lab work
- `speech-representations`: unpushed commits. It also uses LFS, so run `git lfs push --all`.
- The two `gemini-agy-*` repos: LOCALREMOTE (their only remote is a local path, about 9 commits each).
  Give them a GitHub remote.

### 3. Remove the stale laptop copies of lab-owned repos
- Clean and identical on both hosts, so they can go once the audit row is clean: `complex-seizures`,
  `llm-therapy`, `galleria`, `yjmantilla.github.io`.
- Big ones; check their ignored and LFS data against the lab first:
  `containerized-red-teaming` (31 GB ignored, 24 behind, LFS), `deprecated-speech-representations`
  (7.9 GB, LFS).
- Undecided (`[?]` in projects.yaml; branches differ per host): `neurodags`, `neurodags-private`,
  `cocofeats`, `coco-pipe`. Pick an owner for each, then treat them like the others.

### 4. Epilepsy (its own step)
- `deprecated-epilepsy` holds 67 GB of ignored data, plus the laptop's prereg caches and fifs. Move them to the lab
  with the README's staged rsync and `rsync -anci` verification. Then set `epilepsy` and `deprecated-epilepsy` to
  `[lab]` in projects.yaml.

### 5. Finish `~/code`
- Move what remains into `~/Projects`. For each third-party clone (SWE-bench, langchain,
  schnelle-umlaute, waydroid_script, NeuripsEEG2025, PsychoBench, …), re-clone it or delete it.
- Delete `~/code` once fleet-audit shows no `LOOSE` row for it.

## Small fixes
- Laptop `cocofeats`: the local branch `dev` blocks fetching `dev/0.4.0` (ref name clash). Rename or delete it.
- Lab: a stray `~/code` (empty `epilepsy` dir, plus `neurodags-private`). Clean it up during step 3.
- Overleaf repos show FETCHFAIL, because the audit has no credentials. Check them by hand.
- Move the lab's cluster-utilities checkout from the `fleet` branch to `main`.

## Rules (the same as everywhere else in fleet/)
- Ask before any push or delete.
- Only forge remotes count as a backup (GitHub, GitLab, Overleaf, Bitbucket, Codeberg). Another fleet host
  does not.
- An `LFS-EXTERNAL` flag means keep that copy.
