# kjerbi lab — Alliance allocations (CCDB snapshot)

Copied from CCDB → *My Account → My Resources and Allocations* on **2026-10-01**. PI: Karim Jerbi (kif-392). Re-check CCDB for
current figures; RAC awards renew yearly (the current ones expire 2027-04-01).

## RAC — `rrg-kjerbi` (RAPI kif-392-ac)

| Resource | Allocation | Type | Expires | Slurm account |
|---|---|---|---|---|
| Fir | **715 core-years** (715 core-equivalents, 2859 GB total system memory) | CPU | 2027-04-01 | `rrg-kjerbi_cpu` |
| Fir | 50 TB project storage + 299 TB nearline | Storage | 2027-05-01 | — |
| Trillium | **150 RGU-years** (157.38 RGU-equivalent years, 2360.7 GB memory, 49.18 CPU cores) | GPU | 2027-04-01 | `rrg-kjerbi` |
| Trillium | 150 TB project storage | Storage | 2027-05-01 | — |

Conversions: 1 core-year = 8,760 core-hours, so 715 core-years ≈ **6.26M core-hours per allocation year** on fir. That is the target
*rate* the scheduler's fair share is tuned to: an average of ~715 cores in continuous use. It is not a hard cap. The award is shared
by the whole group, so ask the PI what share a project may use.

Fair share as seen from fir: `sshare -l -A rrg-kjerbi_cpu` shows RawShares 715000, against 104393 for `def-kjerbi_cpu`.

## Default (RAS) — `def-kjerbi` (RAPI kif-392-aa)

On Narval, Nibi, Rorqual, Fir and Trillium: **1 TB project storage each**, plus *opportunistic* CPU/GPU use at low priority, with no
guaranteed amount. Slurm accounts are `def-kjerbi` / `def-kjerbi_cpu` / `def-kjerbi_gpu` depending on the cluster.

## AI (PAICE) — `aip-kjerbi` (RAPI kif-392-ab)

GPU access on **TamIA**, **Vulcan** and **Killarney**, expiring 2030-04-01. CCDB shows no numeric allocation. See the per-cluster docs
in `compute-canada/`.
