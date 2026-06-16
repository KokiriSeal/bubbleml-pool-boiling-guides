# Running Pool Boiling — WallSuperheat-FC72-2D on Lambda

Instructions for running the **WallSuperheat-FC72-2D** study, a pool-boiling
parametric sweep where each `Twall-XX/` directory is one simulation at a
different heater wall temperature.

> **Prerequisite:** the software stack from `LAMBDA_SETUP.md` must already be
> built on this instance (HDF5, AMReX 23.11, `software/Flash-X`, the
> `sites/lambda` site, `config.sh` edited, root `Jobfile` set to `bash`). This
> guide reuses all of that — it does **not** rebuild AMReX/HDF5.

Instance IP: `132.145.142.183` · SSH key: `BubbleID.pem`

---

## Read first — how this study differs from SingleBubble

1. **One run per wall temperature.** You submit a specific subdirectory, e.g.
   `simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-100`, not the parent.
   Jobrunner merges the parent `flash.toml` (shared physics) with the
   `Twall-XX/flash.toml` (that run's specifics).

2. **It ships as a RESTART run and will fail as-is.** The configs set
   `restart = ".true."` and expect a checkpoint file
   (`INS_Pool_Boiling_hdf5_chk_0020`) that is **not in the repo**.
   `flashRestart.sh` also tries to copy a `chk_0000` that doesn't exist. We
   convert it to a **fresh start** below.

3. **Much bigger and slower.** Grid is `32×32` blocks × `16×16` cells =
   **512×512** (SingleBubble was 96×288), with 2–140 nucleating bubbles and
   `tmax=200`. On CPU this can take hours to days, so we cut `tmax` for testing.

4. **Different pinned Flash-X commit** (`ef1d9026`) — inaccessible like
   `afee52b0`, so we strip the `git checkout`.

### Choosing a wall temperature

Higher `Twall` → hotter wall → more bubbles → slower:

| Dir | Nucleation sites | Stefan # | Notes |
|---|---|---|---|
| `Twall-60` | 2 | 0.027 | gentlest, fastest — **recommended first test** |
| `Twall-100` | 40 | 0.558 | canonical mid-range case |
| `Twall-200` | 140 | 1.886 | most violent, slowest |

---

## Step 1 — Choose your run

```bash
cd ~/Outflow-Forcing-BubbleML-main
TWALL=Twall-60      # change to Twall-100, Twall-200, etc.
SIM=simulation/PoolBoiling/WallSuperheat-FC72-2D
```

## Step 2 — Fix `flashOptions.sh`

Apply the same flag fixes as SingleBubble, on a single line.

```bash
echo 'FlashOptions="incompFlow/PoolBoiling -auto -maxblocks=100 +amrex +parallelIO -site=$SiteHome -makefile=FlashX InsForceInOut=True InsLSDamping=False InsExtras=True IOWriteGridFiles=True -2d -nxb=16 -nyb=16"' > $SIM/flashOptions.sh
```

> Renames `SimForceInOut`→`InsForceInOut`, adds `InsLSDamping=False` and
> `InsExtras=True` (defines `omgm`), and removes the `\` line continuation that
> jobrunner mis-parses.

## Step 3 — Fix `flashBuild.sh`

Remove the inaccessible commit checkout.

```bash
sed -i 's/cd $FLASHX_HOME && git checkout ef1d9026 && .\/setup/cd $FLASHX_HOME \&\& .\/setup/' $SIM/flashBuild.sh
```

## Step 4 — Neutralize the restart step

We start fresh, so make `flashRestart.sh` a no-op (otherwise it aborts trying to
copy a missing checkpoint).

```bash
echo '# fresh start - no checkpoint to restore' > $SIM/flashRestart.sh
```

## Step 5 — Set the MPI process count

512×512 = 1024 blocks; 32 divides it cleanly (32 blocks/rank, under
`maxblocks=100`).

```bash
sed -i 's/^\tmpirun job.target/\tmpirun -np 32 job.target/' $SIM/flashRun.sh
grep mpirun $SIM/flashRun.sh    # should show: mpirun -np 32 job.target
```

If `grep` still shows bare `mpirun job.target`, edit `$SIM/flashRun.sh` and
change the line under the `else` branch to `mpirun -np 32 job.target` by hand
(the leading whitespace can vary).

## Step 6 — Convert the chosen run to a fresh start + short test time

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
sed -i 's/Parfile.restart = ".true."/Parfile.restart = ".false."/' flash.toml
sed -i 's/Parfile.checkpointFileNumber = 20/Parfile.checkpointFileNumber = 0/' flash.toml
sed -i 's/Parfile.plotFileNumber = 100/Parfile.plotFileNumber = 0/' flash.toml
sed -i 's/Parfile.tmax = 200.0/Parfile.tmax = 10.0/' flash.toml   # short test; raise later
cat flash.toml    # sanity-check
```

> For `Twall-60` the checkpoint/plot numbers are already `0`, so those two
> `sed`s simply no-op — harmless.

## Step 7 — Build

```bash
cd ~/Outflow-Forcing-BubbleML-main
rm -rf software/Flash-X/object
jobrunner setup $SIM/$TWALL
```

Expect `success`.

## Step 8 — Generate inputs + fix the heater filename

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
python3 ../../../flashInput.py --input flash.toml   # note: three levels up
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001
```

> Heater name is `pool_boiling`, so the same filename mismatch as SingleBubble
> applies — copy it to the generic `flash_hdf5_htr_0001`.

## Step 9 — Run

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner submit $SIM/$TWALL
```

`Submitting job` with no further output is normal. Monitor from a second SSH
session:

```bash
DIR=~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/$TWALL
tail -f $DIR/INS_Pool_Boiling.log
ls $DIR/*plt_cnt* 2>/dev/null | wc -l
ps aux | grep flashx
```

---

## Converting this run to a BubbleML HDF5 file

Same process as in `LAMBDA_SETUP.md` (Phase 16) — only the source path changes.
Edit the `sim_dir` line in `~/BubbleML-main/scripts/convert.py`:

```python
sim_dir = "/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-60"
output_file = "/home/ubuntu/WallSuperheat-Twall-60.hdf5"
```

Then:

```bash
cd ~/BubbleML-main/scripts
python3 convert.py
```

> Because WallSuperheat varies the wall temperature across runs, this is a study
> where you should **re-dimensionalize** temperature (multiply the normalized
> `temperature` field by the heater temp) before comparing different `Twall`
> runs — see `bubbleml_data/DOCS.md`. For a single run viewed on its own, the
> normalized [0–1] field is fine.

---

## Parameter reference (WallSuperheat vs SingleBubble)

| | SingleBubble | WallSuperheat |
|---|---|---|
| Domain | 2 × 6 | 16 × 16 |
| Resolution | 96 × 288 | 512 × 512 |
| Blocks (2D) | 6 × 18 = 108 | 32 × 32 = 1024 |
| MPI ranks used | 12 | 32 |
| Nucleation sites | 1 | 2–140 (by Twall) |
| Reynolds (1/invRe) | ~303 | ~238 |
| Prandtl | 7 | 8.4 |
| Output cadence | every 0.5 | every 1.0 |
| Shipped as | fresh run | restart run (converted to fresh) |

---

## Summary of changes vs the stock repo (this study)

| File / action | Change | Reason |
|---|---|---|
| `flashOptions.sh` | `Ins*` flags, `+InsExtras=True`, single line | renames, `omgm_var`, parse bug |
| `flashBuild.sh` | drop `git checkout ef1d9026` | commit inaccessible / not a git repo |
| `flashRestart.sh` | emptied to a no-op | missing checkpoint would abort submit |
| `flashRun.sh` | `mpirun -np 32 job.target` | bare mpirun segfaults; 32 divides 1024 |
| `Twall-XX/flash.toml` | `restart=.false.`, ckpt/plot `0`, `tmax=10` | fresh start + short test run |
| heater file | copy to `flash_hdf5_htr_0001` | filename mismatch |
