# Compute Canada (Alliance) + Claude Code — Operating Guide

**Purpose.** A project-agnostic handoff for working on Digital Research Alliance of Canada (Compute Canada) HPC clusters — `fir`, `narval`, `beluga`, `graham`, `cedar`, `trillium` — with a coding agent (Claude Code or similar). It captures the cluster workflow, safe-automation rules, and the gotchas that cost real debugging time, so you don't have to rediscover them.

> **How to use this.** Read Parts A–B to get oriented. If you use Claude Code / a coding agent, drop Part A + Part B into your own **global** `~/.claude/CLAUDE.md` so the agent inherits the rules on every project (see Part D). Everything here is generic Alliance knowledge — **no project-specific code, data, or results** — fill in the placeholders below for your own setup.

> **⚠️ Fill these in for your own setup.** Replace `<username>` with your Alliance/CCDB username everywhere. Replace `<group>` / `<lab>` with your allocation group. Use **your own** SSH keys and credentials — never reuse anyone else's. Confirm with your PI which allocation you're entitled to (`def-<lab>`, `rrg-<lab>`, `def-<lab>_gpu`).

---

## TL;DR quick-start

1. **Never run compute on a login node.** Login nodes are for editing, `git`, `squeue`/`sbatch`/`sacct`, `rsync`, and small `ls`/`cat` only. Everything heavier (`pip install`, `pytest`, `import jax/torch/mne`, any fit/training/preprocessing) goes inside `salloc` (interactive) or `sbatch` (batch).
2. **From Windows, drive the cluster through WSL, not git-bash.** WSL `ssh` supports connection multiplexing (ControlMaster) so you authenticate Duo once and reuse the socket. (On macOS/Linux, native `ssh` multiplexing works directly.)
3. **Compute on the cluster, analyse on your laptop.** `rsync` results back and run plotting/aggregation locally — never on a login node.
4. **Smoke-test cheap before you spend big.** Run one representative job, check it with `seff`, then scale. Use the smoke run to choose the *shape* too — array vs serial loop vs within-node parallel (B.4) — not just to prove the code runs. Gate expensive jobs behind a passing smoke test.
5. **Nothing irreversible without a human yes.** Show the exact command before any `sbatch`, `git push`, or file deletion, and wait for approval.

---

## Setup — put `cluster-login` on your PATH

`cluster-login` (this repo's `bin/`) is the one helper worth installing — the "MFA refresh button" that (re)opens the shared SSH master. Agents don't need a wrapper to *reach* the cluster: they call `ssh -o BatchMode=yes <host> '<cmd>'` directly (rule A.6). (`bin/cluster-run` is **deprecated** — now just a stub that redirects you to that `ssh` form.)

When `ssh -o BatchMode=yes <host>` fails or a job vanishes, run **`cluster-status [host …]`** (`bin/`) FIRST — it probes each host's master + live reachability so you can tell *your* auth expired (→ `cluster-login <host>`) from *the cluster* being down/slow, and prints the Alliance status URL (https://status.alliancecan.ca/, JS-rendered, no stable JSON API) for incidents/maintenance. e.g. it distinguishes a working `trillium` login node from a down `trillium-gpu` one.

To check storage, run **`cluster-quota [host …]`** (`bin/`). It runs `diskusage_report` on every cluster in parallel (default: fir, narval, rorqual, nibi, trillium) and prints one table of space **and** file-count usage per filesystem and allocation. Anything at ≥80% is flagged, and anything over quota is flagged OVER. `--only rrg-kjerbi` narrows it to one allocation's project/nearline rows. It's read-only and light enough for a login node.

**Recommended — symlink it into a dir already on your PATH** (e.g. `~/.local/bin`):
```bash
ln -s "$PWD/bin/cluster-login" ~/.local/bin/cluster-login
```
The symlink (not a copy) means edits to the script and `git pull`s are picked up automatically. Ensure the target dir is on your PATH: fish `fish_add_path ~/.local/bin`; bash/zsh add `export PATH="$HOME/.local/bin:$PATH"` to your rc file.

**Alternative — put the whole `bin/` on your PATH:** fish `fish_add_path "$PWD/bin"`; bash/zsh `export PATH="$PWD/bin:$PATH"` in your rc.

A **new** shell picks it up immediately; in an **already-open** shell run `rehash` (fish/zsh) or `hash -r` (bash) once. Undo: `rm ~/.local/bin/cluster-login`.

---

## Part A — Operating rules for a coding agent on the cluster

Generic, safe-automation rules. These keep an agent from doing something expensive or destructive without you in the loop. Good defaults for any HPC project.

### A.1 Stop and ask before anything irreversible or outward-facing
Require explicit approval before:
- Submitting Slurm jobs that do real compute (`sbatch` / heavy `salloc` / `srun`)
- `git push` (any remote) — and **never** force-push
- Deleting result or data files
- Any push to an external service (Overleaf, a paper repo, a shared doc)

And flag immediately (don't just continue) if results contradict earlier conclusions or a memory/runtime regression appears.

### A.2 No autonomous loops, no persistent agents on login
- One compute cycle per explicit approval — no self-driving loops.
- Never start a persistent agent, server, Jupyter, or long-lived session **on a login node**.
- Show the exact command and wait — every time.

### A.3 Secrets & hygiene
- Never commit or paste secrets (SSH keys, Duo, API tokens, git remote URLs that embed a token).
- Keep code and large result files separate; bulky outputs go to `/scratch`, release artifacts, Git LFS, or ignored storage — not your code repo.

### A.4 Before any `sbatch`, show the plan
The agent should print, and you should sanity-check: **account · partition/GPU request · time · memory · CPUs · expected output files · the exact command.** Cross-check against `seff` from a prior run.

### A.5 Report faithfully
If a job fails, say so with the output. If a step was skipped, say it. Don't claim a result is "validated" or "done" unless the evidence is in hand. (If you write up results, prefer cautious language — "preliminary", "in this configuration", "suggests" — over absolute claims.)

### A.6 Reach the cluster with `ssh -o BatchMode=yes`, never a bare interactive ssh
An agent's cluster calls must **never** be able to hang on a Duo prompt no one can answer. A fresh/interactive `ssh` triggers Duo MFA and **hangs**; `-o BatchMode=yes` makes ssh **fail fast** instead:
```bash
ssh -o BatchMode=yes <host> '<cmd>'
```
Master up → it reuses the shared SSH socket and runs instantly, no MFA. Master down → it exits immediately with `Permission denied (...keyboard-interactive)` rather than blocking. On that failure, **stop and tell the user** — they refresh from their phone (`cluster-login <host>` + approve Duo); do **not** retry in a loop. Full setup: `mobile-access.md`.

> `bin/cluster-run` is **deprecated**. It wrapped exactly this (`ssh -O check` + `BatchMode=yes`, exit 42 on master-down), but the wrapper's semantics tripped up agents — use the plain `ssh -o BatchMode=yes` form above. The script is now a stub that just prints the equivalent command.

---

## Part B — Alliance cluster workflow

### B.1 The golden rule: never compute on login nodes
A login node (e.g. `login1.fir.alliancecan.ca`) is shared by everyone and is for light interactive use only.

**Forbidden on login** (trips Alliance abuse detection → throttling/ban):
`pip install` (anything non-trivial — pulls deps, compiles wheels) · `pytest` · `python -c "import jax/torch/mne"` (triggers XLA/CUDA/scipy init) · any fit, training loop, or dataset preprocessing · long repo-importing scripts · background pollers that re-SSH every few seconds.

**OK on login:** `git pull/status/log/push` (small repos) · `squeue`/`sbatch`/`scancel`/`sacct`/`seff`/`sstat` · `ls`/`cat`/`head`/`tail` on small files · `rsync` to/from your laptop · `nano`/`vim` · `module list`/`module avail` · `env`/`which python`.

**If you slip:** `Ctrl+C` immediately; `pkill -u $USER python` if it lingers; drop a dated note in `/scratch/$USER/notes/` so you have a paper trail; use `salloc` going forward.

> A one-shot `ssh cluster 'git pull'` / `squeue` / `sbatch` is fine — it's not a persistent agent. The moment that one-shot command is a Python import or `pip install`, it's compute on login.

### B.2 SSH setup — multiplex so you Duo once
Alliance clusters require **publickey + Duo MFA** on every fresh connection. A non-interactive tool can't answer Duo, so the trick is **connection multiplexing**: authenticate once interactively, then reuse the socket for all subsequent calls.

**On macOS / Linux** — native `ssh` multiplexing works. In `~/.ssh/config`:
```ssh-config
Host *
  ServerAliveInterval 300
  ControlMaster auto
  ControlPersist 4h
  ControlPath ~/.ssh/cm-%r@%h:%p

Host fir
  HostName fir.alliancecan.ca
  User <username>
  IdentityFile ~/.ssh/id_ed25519
```

**On Windows** — **use WSL, not git-bash.** Windows/Git-for-Windows `ssh` cannot multiplex (`mux_client_request_session: read from master failed` / `Failed to connect to new control master`); do **not** put `ControlMaster` in `C:\Users\<you>\.ssh\config`. Instead, put the config above inside **WSL's** `~/.ssh/config` (the `:` in `ControlPath` is legal on the Linux filesystem). Then:

```bash
# keep ONE interactive session open in a WSL terminal — approve Duo once; it's the master socket
wsl ssh fir
# all other calls ride the master with no Duo:
wsl.exe bash -c "ssh fir 'squeue --me'"
# stream a multi-line script (login shell so sbatch/modules are on PATH; quote 'EOF' so $vars expand on the cluster):
wsl.exe bash -c 'ssh fir "bash -l -s"' <<'EOF'
cd /scratch/$USER/myproject && git pull --ff-only
EOF
```
If a call suddenly returns `Permission denied (keyboard-interactive)`, the master expired — run `ssh fir` once interactively to re-establish it.

> **Keys:** `ssh-keygen -t ed25519`, then upload the **public** key via the CCDB portal (https://ccdb.alliancecan.ca → Manage SSH Keys), not by hand-editing `authorized_keys`. A CCDB-registered key does **not** exempt you from Duo on interactive login.

> **Host keys (verifying the cluster, not yourself):** on a first connect ssh asks *"authenticity of host … can't be established"* — don't just type `yes`; compare the fingerprint against an out-of-band source. Same when ssh warns `REMOTE HOST IDENTIFICATION HAS CHANGED`, which distinguishes a rebuilt login node from an interception. Published fingerprints for every Alliance cluster are clipped into **`SSH host keys.md`** in this repo; the wiki (https://docs.alliancecan.ca/wiki/SSH_host_keys) stays authoritative if a cluster rotates its keys. Check a live node without logging in (no auth, no MFA): `ssh-keyscan -t ed25519 <host> | ssh-keygen -lf -`.

> **Tip:** don't pass complex nested-quote command strings through `wsl ssh`. Write a clean `.py`/`.sh` locally, `scp` it over, and run it.

> **Agents:** drive the cluster with `ssh -o BatchMode=yes <host> <cmd>` — it reuses the master and fails fast (`Permission denied`) instead of hanging on Duo when the master is down; refresh with `cluster-login <host>`. (`cluster-run` is deprecated — see A.6.) For a long-lived, headless setup prefer `ControlPersist yes` over a fixed timeout so the master expires only on a real disconnect. See `mobile-access.md` + rule A.6.

### B.3 Interactive work: `salloc`
```bash
# CPU
salloc --time=1:00:00 --cpus-per-task=4 --mem=16G --account=def-<lab>
# GPU (fir H100)
salloc --time=1:00:00 --cpus-per-task=4 --mem=32G --gres=gpu:h100:1 --account=def-<lab>_gpu
srun --pty bash -l        # land in the compute-node shell
hostname                  # confirm a node like fc10408, NOT loginN
# now pip install / pytest / heavy imports / GPU sanity-checks are all fine
exit                      # release the allocation ASAP
```

### B.4 Batch work: `sbatch`
Long jobs always go through `sbatch`, never `salloc` + manual run. **Array jobs are a good default for independent, *long-running* tasks** — easy reruns, cleaner logs, better scheduling — but they are not automatically the right shape. Pick the shape before you scale (see below).

Header cheat sheet:
```bash
#!/bin/bash
#SBATCH --job-name=descriptive_name
#SBATCH --account=def-<lab>           # or rrg-<lab> when the job truly uses the CPU/GPU
#SBATCH --time=HH:MM:SS               # shorter realistic requests schedule faster
#SBATCH --cpus-per-task=N             # match true parallelism
#SBATCH --mem=XG                      # enough, not wildly high
#SBATCH --gres=gpu:h100:1             # only when GPU-bound
#SBATCH --output=logs/%x-%A_%a.out
#SBATCH --error=logs/%x-%A_%a.err
#SBATCH --array=0-99%20               # %20 caps concurrent tasks (see shape guidance below)
```

Chain stages with dependencies — gate arrays behind a smoke test so a broken env fails cheap:
```bash
SMOKE=$(sbatch --parsable submit_smoke.sh)
JOB=$(sbatch --parsable --dependency=afterok:$SMOKE submit_array.sh)
```
`afterok` = run only if the parent succeeded; `afterany` = run regardless; `--parsable` = print just the job id.

**Choose the shape: array, serial loop, or within-node parallel.** An array is *not* automatically fastest. Every array task re-pays the full startup cost — queue wait, node allocation, `module load`, venv activation, interpreter start, heavy imports (`torch`/`jax`/`mne`), model/dataset load, cache warm. When that overhead is comparable to the actual per-item work, an array of many small tasks is **slower and more expensive** than one job looping serially.

Measure two numbers in the smoke run, then pick:

| Per-item compute `T_item` vs startup `T_start` | Best shape | Why |
|---|---|---|
| `T_item ≫ T_start` (~10×+) | **Array**, one item per task | Overhead amortizes; spreads across nodes; independent retries |
| `T_item ≲ T_start` | **One job, serial loop** (or chunked array) | Pay setup once instead of N× re-import and N× queue waits |
| Many short items, fits one node | **Within-node parallel** (`joblib`/`multiprocessing`/GNU `parallel`) | Setup once, then saturate all `--cpus-per-task` cores |

**Chunking is the usual middle ground** — an array whose tasks each loop over a slice, so you get parallelism *and* amortized setup:
```bash
#SBATCH --array=0-9                   # 10 tasks × 50 items = 500 items
python run.py --shard "$SLURM_ARRAY_TASK_ID" --num-shards 10
```

Other pulls away from one-item-per-task arrays: **shared warm state** (load a big model/dataset once per job, not once per task — also avoids concurrent-write races on a shared cache); **GPU packing** (a whole H100 per tiny task badly under-uses a measured allocation — see B.6); **fairshare** (thousands of tiny tasks cost scheduling overhead and queue time).

> Smoke-test the **shape**, not just the code. Run one representative item and `seff` it — if `Elapsed` is dominated by startup rather than compute, restructure *before* launching the array.

### B.5 Right-size resources (don't guess)
Run **one** representative job, then:
```bash
seff <job_id>     # one-line efficiency summary
sacct -j <job_id> --format=JobID,State,Elapsed,TotalCPU,AllocCPUS,MaxRSS,ReqMem
```
Set `--mem` ≈ `MaxRSS` × ~1.3, `--time` ≈ `Elapsed` × ~1.5, `--cpus-per-task` to the parallelism you actually use. If `MaxRSS` ≪ `ReqMem`, you over-requested — it slows your future scheduling (fairshare). GPU jobs are usually short and memory-light per task; CPU jobs of the same work run much longer and need more RAM — size them separately.

### B.6 Which cluster, and which account
| Use case | Cluster | Why |
|---|---|---|
| Feature extraction, preprocessing, dataset conversion, metrics, small/medium CPU jobs | **fir** | Don't burn a GPU allocation unless the work is actually GPU-bound |
| Single-GPU training/inference that fits on one H100 | **fir GPU partition** | Faster queue than Trillium for small GPU jobs |
| Foundation-model training, large multi-GPU / multi-node | **Trillium-GPU** | Only when genuinely GPU-bound and large |

- **`def-<lab>`** — default allocation, modest priority. Use when you can't yet prove the CPU/GPU is well-utilised.
- **`rrg-<lab>`** — higher-priority RRG allocation. Reserve for jobs that strongly use the CPU/GPU; don't spend it on exploratory work.
- fir GPU partition: `--account=def-<lab>_gpu --partition=gpubase_bygpu_b1 --gres=gpu:h100:1`.
- **Trillium GPU**: `--account=rrg-<lab> --nodes=1 --gpus-per-node=1` (no `_gpu` suffix, no `--gres`/`--mem` — see Part C). A single GPU is a **whole H100 (80 GB)** — no MIG — so a small job (one modest model) badly under-uses it, and **GPU utilisation is measured** on the RAC. Pack the GPU: run several processes concurrently in one job (variants/seeds/folds), optionally under **CUDA MPS** (`nvidia-cuda-mps-control -d`) for true co-execution. Pre-warm any shared cache in one process first to avoid concurrent-write races.

### B.7 Storage hierarchy
| Path | Use | Notes |
|---|---|---|
| `/home/$USER` | small code, env files, shell config | ~50 GB, backed up; **may be read-only from compute nodes** — don't write job outputs here |
| `/project/<group>` | shared raw + processed data, final checkpoints, shared outputs | shared group quota; long-lived artifacts |
| `/scratch/$USER` | working space, intermediates, job outputs | ~20 TB; **purged after 60 days** — not for anything you want to keep |
| `$SLURM_TMPDIR` | node-local hot data during a running job | gone when the job ends |

Move caches off `$HOME`: `export XDG_CACHE_HOME=/scratch/$USER/.cache; export PIP_CACHE_DIR=/scratch/$USER/.cache/pip`. Checkpoint regularly. Code lives in git, not as the source of truth on the cluster.

### B.8 Building environments & datasets
- **Build venvs / install packages inside an allocation** (`salloc`), never on login. On Alliance, prefer the module stack + `virtualenv --no-download`, then `pip install --no-index` against the wheelhouse where possible.
- **Internet access on compute nodes is cluster-dependent.** Do not assume — test with a one-liner job (`sbatch --wrap="curl -s --max-time 10 https://example.com && echo OK || echo FAIL" ...`). Known state as of 2026:
  - **fir**: compute nodes **have** outbound internet — HuggingFace, pip, wget all work from batch jobs.
  - **Trillium**: compute nodes have **NO** internet (only the GPU login node `trig-login01` and OnDemand apps do). Pre-cache models/wheels on the login node, then run offline (`HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1`, point `HF_HOME` at a pre-populated `$SCRATCH/hf_cache`).
  - **Older clusters (Cedar, Graham, Beluga)**: compute nodes typically have **no** internet. Download on the login node or a data-transfer node (DTN) instead.
  - When in doubt, test first — don't assume either way.
- **Download jobs** (HuggingFace, wget, git-annex, rsync from external): these are I/O-bound, not compute-bound. Use minimal resources — don't waste GPU allocation or over-request CPUs and memory:
  ```bash
  #SBATCH --account=def-<lab>       # CPU account, NOT the _gpu account
  #SBATCH --cpus-per-task=2         # download is I/O-bound, not CPU-bound
  #SBATCH --mem=8G                  # enough for HF snapshot + light preprocessing
  #SBATCH --time=24:00:00           # generous; large datasets can be slow
  ```
  If the cluster has no internet, use the login node for the download step only (`ssh cluster 'cd /scratch/... && huggingface-cli download ...'`), then submit a separate compute job for preprocessing.
- **DataLad / git-annex datasets**: fetch only what you need:
  ```bash
  module load git-annex/<version>
  cd /project/<group>/datasets/<dataset>
  git annex get <only the files you need>
  ```
  Unfetched annex files appear as **broken symlinks** — a job that hits one fails fast with `FileNotFoundError`. Verify what's actually materialised before a sweep: `find -L <dir> -name "<pattern>" -type f` (`-L` follows symlinks; `-type f` drops the broken ones).
- **Upload throughput is dominated by *your* location, not the cluster.** The same `rsync` to `/scratch` is fast from a well-connected on-campus link and noticeably slower from home or a lab. Measured 2026-09-04 from **Mila**: ~1.5 GB per subject in ~2 min (~13 MB/s effective, post-compression) into Fir `/scratch`; the same transfers are much slower from home/lab. Only measured on Fir so far, but this is a client-link effect and likely **general to Alliance clusters**. For large datasets, transfer from the fastest link you have, or use **Globus** (per-cluster endpoint), which is far less sensitive to your local connection. `rsync`/`scp` go through the login node.

### B.9 The canonical agent-driven loop
1. **Local:** code, test, commit, push to origin.
2. **One-shot SSH (login-safe):** `ssh cluster 'cd /scratch/$USER/repo && git pull --ff-only'`
3. **Submit from login:** `ssh cluster 'cd repo && sbatch --array=… submit.sh'` — capture the job id.
4. **Poll occasionally** (≤ once/min, one `sacct`, don't keep a session warm): `ssh cluster 'sacct -j <id> --format=State,Elapsed,MaxRSS,ExitCode -P | head'`
5. **When done:** `rsync -av cluster:/scratch/$USER/results/ ./results/`
6. **Analyse locally.** The cluster is compute-only.

### B.10 Monitoring & cleanup
```bash
squeue -u "$USER"            # active jobs
seff <id>                    # efficiency summary after a run
sstat -j <id>                # live stats on a running job
diskusage_report             # /home, /project, /scratch quotas
du -sh /scratch/$USER/*      # find bloat
```
Never delete shared files without checking ownership and the project README. Archive inactive data to `/nearline`; don't bulk-delete.

---

## Part C — Gotchas catalogue (cost real time; now you skip them)

| Symptom / situation | What's actually going on | Fix |
|---|---|---|
| `ssh cluster` denied from Windows git-bash; `mux_client_request_session: read from master failed` | Windows `ssh` can't multiplex | Use **WSL** ssh with ControlMaster; keep one interactive session open as the master (Part B.2) |
| GPU job: `Unable to initialize backend 'cuda'` → with `JAX_TRACEBACK_FILTERING=off`, `cuInit(0) failed: CUDA_ERROR_NO_DEVICE`, **yet `nvidia-smi` shows the GPU** | A **bad GPU node**, not your env | `--exclude=<node>` and resubmit. Bad nodes change over time — diagnose by the symptom, exclude, retry. Don't start editing your CUDA env/`LD_LIBRARY_PATH` over this. |
| `sbatch: error: Batch script contains DOS line breaks (\r\n)` | A `.sh`/`.sbatch` got saved with Windows CRLF | Cluster-bound files need **LF**. From Windows Python: `Path.write_text(text, newline="\n")`. Quick patch on the cluster: `sed -i 's/\r$//' <file>`. Lock it with `.gitattributes`: `*.sh text eol=lf` / `*.sbatch text eol=lf`. |
| A `.ps1` script fails on Windows with "string is missing the terminator `"@`" | Windows **PowerShell 5.1** requires CRLF for here-strings (PowerShell 7 is fine with either) | Windows-bound `.ps1` need **CRLF** — the opposite of cluster `.sh`. `.gitattributes`: `*.ps1 text eol=crlf`. |
| `sbatch: NOTE: Your memory request of 32768.0M was likely submitted as 32.0G…` | **Informational only** — Slurm reads `G` as binary (1024 M); your `--mem=32G` is honoured as intended | Ignore it. Don't `2>/dev/null` (hides real errors); don't switch to `--mem=32000M` (that changes the actual allocation). |
| `pip install jax[cuda12]` "downgrades" JAX and you end up CPU-only | JAX CUDA wheels are **Linux-only**; there's no native-Windows GPU wheel, so pip strips the extra | Don't install the cuda extra in a native Windows venv. Use WSL2 (Linux), or the cluster. Cluster GPU install pattern: `pip install -e ".[gpu]" -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html`. On WSL2, install the **Windows** NVIDIA driver only — never a Linux display driver inside WSL. |
| HuggingFace / wget download fails inside a batch job | Compute nodes may not have internet — depends on the cluster (see B.8) | Test first: `sbatch --wrap="curl -s https://example.com && echo OK || echo FAIL" ...`. On **fir**, compute nodes have internet and downloads work fine in batch jobs. On older clusters (Cedar/Graham/Beluga), use the login node or a DTN. |
| Download job uses 8 CPUs, 32G, GPU account — queues for hours | Downloading is I/O-bound; over-requesting burns fairshare and delays scheduling | Use `--cpus-per-task=2 --mem=8G --account=def-<lab>` (CPU account). See B.8 download job template. |
| Launched a big **array job**, then had to cancel it — a plain serial `for` loop was the better call | Per-task startup (queue wait, `module load`, venv, heavy imports, model/data load) dominated the small per-item compute, so the array paid that cost **N×** instead of once | Compare per-item compute against per-task startup in a smoke run **before** choosing the shape (B.4). When startup ≳ per-item work, use a serial loop, within-node parallelism, or a **chunked** array (`--array=0-9`, 50 items each) rather than one task per item. |
| WSL2 GPU won't attach: `cuInit failed: CUDA error 100` / "GPU access blocked by the operating system" | Host-level GPU-partitioning (GPU-P) hiccup — Hyper-V can't hand the GPU to the WSL VM (common on laptop dGPUs) | A full **Windows reboot** re-enrolls GPU-P. Beware: `wsl --shutdown` can make it worse and may need `gpuSupport=false` in `%USERPROFILE%\.wslconfig` just to boot WSL. For serious GPU work, use the cluster. |
| Login succeeds at Duo but the shell drops with "Login is offline / Noeud de connexion est hors-ligne" (esp. Narval) | One of several gates **beyond** Duo | Check, in order: (1) a cluster outage at status.alliancecan.ca; (2) the **CCDB access agreement** for that cluster at ccdb.alliancecan.ca/me/access_systems (independent of MFA — every new user must accept it once); (3) a single bad login node (the hostname is round-robin DNS — try `cluster1/2/3` individually). |
| SSH refused: **"System is going down. Unprivileged users are not permitted to log in anymore"** (`pam_nologin(8)`) | The login node you landed on is **rebooting / draining**. systemd writes `/run/nologin` *before* it stops sshd, so you still authenticate and are only then refused. Nothing wrong with your account, keys, or MFA — and it's that one node, not the cluster | The alias is round-robin across several nodes, so another is usually healthy. List them: `getent ahostsv4 <cluster>.alliancecan.ca`. Probe without authenticating (no MFA, not a login attempt): `timeout 5 bash -c 'echo > /dev/tcp/<ip>/22'` or `ssh-keyscan <ip>`. Once the bad node is **fully** down it refuses TCP and ssh skips to the next address automatically — just retry `cluster-login`. If it's mid-drain (TCP still open, ssh won't skip), pin temporarily in `~/.ssh/config` with `HostName <healthy-ip>` + `HostKeyAlias <cluster>.alliancecan.ca` (keeps host-key checks and the shared master working), and revert when maintenance ends. Confirm the window at status.alliancecan.ca. |
| `module load … | tail` then "module not loaded" | The pipe runs the Lmod shell function in a **subshell**, so the load doesn't persist | Don't pipe `module load`. Use redirects (`>/dev/null 2>&1`) or none, then check `module list`. Also: for `srun`, **omit** `--partition` and let account+gres route; `sbatch` accepts the partition fine. |
| **Trillium** `sbatch`: `option --gres not recognized` / `The --mem... options are not allowed on Trillium` | Trillium schedules **whole GPUs or whole nodes** (no MIG) and its submit filter rejects `--gres` and `--mem` | Request GPUs with `--gpus-per-node=1` (¼ node = 24 cores + 188 GiB) or `=4` (whole node); **never** `--gres`/`--mem`. Always `--nodes=1`. Don't hardcode the `compute` partition (scheduler picks it); `-p debug` is OK for tests. Match the docs' single-GPU example exactly. |
| **Trillium**: job runs with **default** parameters, ignoring the env vars you set on the `sbatch` line (`VAR=x sbatch …`) | Trillium's wrapper forces `--export=NONE`, so the submitting shell's env (including `$USER`) does **not** reach the job | Pass them explicitly: `sbatch --export=ALL,VAR=x,VAR2=y script.sh`. Values with spaces are fine inside one quoted `--export`. Without `--export=ALL`, even `$USER`/`$SCRATCH`-derived paths can come up empty. |
| **Trillium** debug job: `Requested time limit is invalid (missing or exceeds some limit)` | The **debug** GPU partition caps walltime at **2 h** (1 GPU); your script's `--time` is longer | Override on the CLI for tests: `sbatch -p debug --time=00:30:00 …`. Use `debugjob -g 1` for a 2 h interactive GPU session instead. |
| Alliance wheelhouse: `ImportError: cannot import name 'sph_harm' from 'scipy.special'` (via `import mne`) | The wheelhouse's newest **mne** (1.9.0) imports `sph_harm`, which **scipy ≥1.16 removed** | Pin `scipy==1.14.1` in the venv (still has `sph_harm`, compatible with mne 1.9.0; uv/pip resolves numpy down to a matching 2.2.x that torch still accepts). General lesson: when the wheelhouse caps a pure-Python package at an old version, pin its compiled deps to that era. |
| `rsync` over ssh dies with `error in rsync protocol data stream (code 12)` | You put **`ssh -n`** in rsync's `-e` transport; `-n` redirects ssh stdin to `/dev/null`, which kills rsync's bidirectional data stream | Drop `-n` from `-e "ssh …"`. `-n` is only for *standalone* `ssh cmd` (to stop it eating a loop's stdin), never for the rsync/scp transport. |
| A `while read … < list` loop that runs `ssh`/`rsync` inside **stops after the first item** (rest of the list silently skipped) | `ssh` (and rsync-over-ssh) reads from the loop's stdin and consumes the remaining lines of `list`, so `read` hits EOF | Read the list from a **separate FD**: `while IFS= read -r x <&3; do … ; done 3< list`. (Alternatively `</dev/null` or `-n` on *standalone* ssh calls — but never on the rsync transport, per the row above.) |

---

## Part D — Setting up your own Claude Code for cluster work

Two layers of "memory" make an agent useful across sessions:

1. **`~/.claude/CLAUDE.md` (global, all projects)** — put Parts A + B of this guide here, with your `<username>`/`<group>` filled in. Every session inherits these rules, so the agent won't try to `pip install` on a login node or push without asking.
2. **Per-project `CLAUDE.md` (in the repo root)** — project-specific rules: which Slurm scripts to use, where outputs go, what "done" means for that project, any stop conditions.
3. **The `memory/` directory (auto-memory)** — Claude can persist one fact per file (gotchas, decisions, environment quirks) and recall them later. This is how knowledge like the gotchas in Part C accumulates over time instead of being re-learned each session.

Start small: drop the Alliance rules into your global `CLAUDE.md`, add a short project `CLAUDE.md`, and let the memory grow as you hit (and solve) issues.

---

## Appendix — Alliance onboarding checklist

- [ ] CCDB account; confirm with your PI which allocation group + accounts you can use (`def-<lab>`, `rrg-<lab>`, `def-<lab>_gpu`).
- [ ] Generate an SSH key (`ssh-keygen -t ed25519`); upload the **public** key via CCDB.
- [ ] On first connect to each cluster, verify its **host-key fingerprint** against `SSH host keys.md` (repo root) before accepting — don't blind-`yes` the prompt.
- [ ] Accept the **CCDB access agreement** for each cluster you'll use at ccdb.alliancecan.ca/me/access_systems (blocks login until done — independent of MFA).
- [ ] Set up Duo MFA; on Windows, install **WSL2** and put the multiplexing `~/.ssh/config` (Part B.2) inside WSL with your username. Test `ssh fir`, approve Duo once.
- [ ] Learn the storage layout (Part B.7): code in git, outputs to `/scratch`, shared data in `/project`.
- [ ] Build your environment **inside `salloc`**, never on login.
- [ ] Bookmark: https://docs.alliancecan.ca · https://status.alliancecan.ca · https://ccdb.alliancecan.ca
- [ ] Read Part A before letting an agent touch the cluster.

---

*This guide is generic Alliance + agent operating knowledge, written for reuse across projects. Authoritative live sources: the Alliance docs (docs.alliancecan.ca), your cluster's status page, and your own project's README/CLAUDE.md.*
