# Custom Pool Boiling Runs — "Twall-107 but hotter" (custom Stefan & numSites)

A focused guide for creating **your own** pool-boiling runs by taking the FC72
Twall-107 setup and turning up the heat. In the FC72 world, "hotter" means a
**higher Stefan number** plus **more nucleation sites** — everything else (domain,
grid, fluid properties) stays the same.

> **Prerequisite:** the software stack and the **FC72 Twall-107** run must already
> be working on this instance (see `WALLSUPERHEAT_FULL_SETUP.md`, Parts A & B).
> In particular, the **shared parent** `flash.toml` must hold the FC72 fluid
> properties. This guide only changes the per-run subdirectory file.

- Instance IP: `132.145.142.183` · SSH key: `BubbleID.pem`

---

## What "hotter" changes (and what it doesn't)

| Lives in | Parameter | Changes when hotter? |
|---|---|---|
| **subdir** `Twall-XXX/flash.toml` | `Parfile.mph_Stefan` | **Yes** — higher |
| **subdir** `Twall-XXX/flash.toml` | `Heater.numSites` | **Yes** — more bubbles |
| parent `flash.toml` | fluid props, domain, grid, boundaries | No |
| parent scripts | flashOptions / flashBuild / flashRun / flashRestart | No (set once) |

So a new temperature = **copy the Twall-107 folder, change two numbers.**

---

## Choosing Stefan and numSites

### Stefan number (physically grounded)

Derived from the two authors' FC72 JSONs (Stefan is linear in wall superheat):

- Twall-103 → 0.5873
- Twall-107 → 0.6395

**Formula:  `Stefan = 0.01305 × (Twall − 58)`**
(the 58 °C intercept ≈ FC-72 saturation temperature)

| Target Twall | Stefan (use this) |
|---|---|
| 110 | 0.6788 |
| 115 | 0.7440 |
| **120** | **0.8091** |
| 130 | 0.9396 |
| 150 | 1.2006 |

### numSites (estimate)

Hotter walls nucleate more sites. We have no authors' correlation for this study,
so these are **reasonable estimates** — the exact positions are auto-placed by
`flashInput.py` (a Halton sequence), not from any JSON.

| Target Twall | Suggested numSites |
|---|---|
| 110 | 45 |
| 115 | 55 |
| **120** | **70** |
| 130 | 90 |
| 150 | 120 |

> **Honesty note:** Stefan values are faithful to the authors' FC72 data. The
> site **counts and positions are not** from the authors (no JSON exists for
> these temperatures). Treat these as physically-plausible custom runs, not
> reproductions of a specific published dataset.

---

## Step 1 — Set your target

```bash
cd ~/Outflow-Forcing-BubbleML-main
SIM=simulation/PoolBoiling/WallSuperheat-FC72-2D
SRC=Twall-107          # the working FC72 run we copy from
TWALL=Twall-120        # <-- your target temperature
STEFAN=0.8091          # <-- from the Stefan table
NSITES=70              # <-- from the numSites table
```

> These shell variables only last for the current SSH session. Re-run this block
> if you log out and back in.

## Step 2 — Confirm the parent is in the FC72 world

The parent `flash.toml` is shared by all Twall runs. If you previously reverted
it (e.g. for a repo Twall-200 attempt), re-apply the FC72 values. Harmless to run
even if it's already FC72.

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM
sed -i \
 -e 's/^Heater.advAngle = .*/Heater.advAngle = 45.0/' \
 -e 's/^Heater.nucWaitTime = .*/Heater.nucWaitTime = 0.4/' \
 -e 's/^Heater.xmin = .*/Heater.xmin = -5.25/' \
 -e 's/^Heater.xmax = .*/Heater.xmax = 5.25/' \
 -e 's/^Parfile.ins_invReynolds = .*/Parfile.ins_invReynolds = 0.0043/' \
 -e 's/^Parfile.ht_Prandtl = .*/Parfile.ht_Prandtl = 7.35/' \
 -e 's/^Parfile.mph_rhoGas = .*/Parfile.mph_rhoGas = 0.008687/' \
 -e 's/^Parfile.mph_muGas = .*/Parfile.mph_muGas = 0.02816/' \
 -e 's/^Parfile.mph_thcoGas = .*/Parfile.mph_thcoGas = 0.209/' \
 -e 's/^Parfile.mph_CpGas = .*/Parfile.mph_CpGas = 0.7997/' \
 flash.toml
cd ~/Outflow-Forcing-BubbleML-main
```

Verify:
```bash
grep -E 'muGas|rhoGas|thcoGas|CpGas|Prandtl|invReynolds|advAngle' $SIM/flash.toml
```

## Step 3 — Create the hotter run from Twall-107

```bash
cp -r $SIM/$SRC $SIM/$TWALL

# Remove stale artifacts copied along from the source run
rm -f $SIM/$TWALL/*hdf5* $SIM/$TWALL/flashx $SIM/$TWALL/flash.par \
      $SIM/$TWALL/merged.toml $SIM/$TWALL/INS_* $SIM/$TWALL/*.log 2>/dev/null
```

The copy inherits Twall-107's fresh-start settings (`restart=.false.`,
checkpoint/plot numbers `0`), so you don't need to redo those.

## Step 4 — Set the two "hotter" knobs

Edit the subdir `flash.toml`.

Option 1 — nano:
```bash
nano $SIM/$TWALL/flash.toml
```
| Find | Change to |
|---|---|
| `Parfile.mph_Stefan = 0.6395` | `Parfile.mph_Stefan = 0.8091` (your STEFAN) |
| `Heater.numSites = 37` | `Heater.numSites = 70` (your NSITES) |

Save and exit.

Option 2 — one-liner (uses the variables from Step 1):
```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
sed -i \
 -e "s/^Parfile.mph_Stefan = .*/Parfile.mph_Stefan = $STEFAN/" \
 -e "s/^Heater.numSites = .*/Heater.numSites = $NSITES/" \
 flash.toml
grep -E 'Stefan|numSites|restart|tmax|plotFileInterval' flash.toml   # sanity check
cd ~/Outflow-Forcing-BubbleML-main
```

> Optional — shorten for a first test (full run is `tmax=200`):
> ```bash
> sed -i 's/^Parfile.tmax = .*/Parfile.tmax = 20.0/' $SIM/$TWALL/flash.toml
> ```

## Step 5 — Build

```bash
cd ~/Outflow-Forcing-BubbleML-main
rm -rf software/Flash-X/object
jobrunner setup $SIM/$TWALL
```

Expect `success`.

## Step 6 — Generate inputs (merged) and fix the heater filename

The config is split across parent + subdir, so merge them for `flashInput.py`
(running it on the subdir alone fails with `KeyError: 'name'`):

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
( cat ../flash.toml; echo; cat flash.toml ) > merged.toml
python3 ../../../flashInput.py --input merged.toml
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001
cd ~/Outflow-Forcing-BubbleML-main
```

You should see it report writing the heater file with your `numSites` value
(Halton-placed sites, radius 0.2).

## Step 7 — Run

```bash
jobrunner submit $SIM/$TWALL
```

Monitor from a second SSH session:
```bash
DIR=~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-120
tail -f $DIR/INS_Pool_Boiling.log
ls $DIR/*plt_cnt* 2>/dev/null | wc -l
ps aux | grep flashx
```

> MPI process count is set in the shared `flashRun.sh` (`mpirun -np 16` for a
> 30-core box; 1024 blocks ÷ 16 = 64 blocks/rank). No change needed per run.

---

## Step 8 — Convert to a BubbleML HDF5 file

Edit the two paths in `~/BubbleML-main/scripts/convert.py` to point at this run:

```python
sim_dir = "/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-120"
output_file = "/home/ubuntu/PoolBoiling-Twall-120.hdf5"
```

Then:
```bash
cd ~/BubbleML-main/scripts
python3 convert.py
```

> This study outputs `pres, velx, vely, dfun, temp, dust` — not `mflx/nrmx/nrmy`,
> so `convert.py` must write only the 7 fields (temperature, velx, vely, dfun,
> pressure, x, y + the two runtime-param sets). Including the missing vars
> `KeyError`s.

Download (on your **Mac**):
```bash
scp -i BubbleID.pem ubuntu@132.145.142.183:/home/ubuntu/PoolBoiling-Twall-120.hdf5 ~/Downloads/
```

> Re-dimensionalize temperature for cross-temperature comparison: multiply the
> normalized [0–1] `temperature` field by the wall temp (e.g. 120).

---

## Making yet another temperature

Everything above is parameterized. To make, say, Twall-150:

```bash
TWALL=Twall-150 ; STEFAN=1.2006 ; NSITES=120
```
…then repeat Steps 3–8. Within the FC72 world, that's genuinely all that changes:
**Stefan + numSites.**

---

## Quick reference — what each run needs

| Item | Where | Per-run? |
|---|---|---|
| FC72 fluid props, domain, grid | parent `flash.toml` | No (shared) |
| flashOptions / flashBuild / flashRun / flashRestart fixes | parent scripts | No (set once) |
| `mph_Stefan` | subdir `flash.toml` | **Yes** |
| `Heater.numSites` | subdir `flash.toml` | **Yes** |
| fresh-start flags (restart/ckpt/plot = false/0/0) | subdir `flash.toml` | inherited from Twall-107 copy |
| merged input + `flash_hdf5_htr_0001` | subdir | **Yes** (regenerate each run) |
| `convert.py` paths | BubbleML scripts | **Yes** (edit per run) |
