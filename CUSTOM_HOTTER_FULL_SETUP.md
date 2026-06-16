# Custom Pool Boiling Run ("Twall-107 but hotter") — Full Setup on Lambda

A **fully self-contained** guide: from a fresh Lambda instance to a running
FC72 pool-boiling simulation at a **custom higher wall temperature** (your own
Stefan number + nucleation-site count), then a BubbleML HDF5 file. Nothing is
assumed to be pre-built. Every file edit is shown explicitly.

This takes the authors' FC72 Twall-107 physics as a base and turns up the heat.
In the FC72 world, "hotter" = a **higher Stefan number** plus **more nucleation
sites**; everything else (domain, grid, fluid properties) stays the same.

- Instance IP used below: `132.145.142.183` — replace with yours.
- SSH key: `BubbleID.pem` (in your Mac's current folder).
- Worked example target: **Twall-120** (Stefan 0.8091, 70 sites). Change these
  in Part C to pick a different temperature.

---=

## How to edit files on Lambda (quick primer)

Most edits use the **nano** editor:

1. Open: `nano <path-to-file>`
2. Move with **arrow keys**, edit by typing.
3. Save: **Ctrl+O**, then **Enter**.
4. Exit: **Ctrl+X**.

Where faster, a one-line `sed` alternative is given. Use whichever you prefer —
**not both**.

---

## Choosing Stefan and numSites (read first)

### Stefan number (physically grounded)

Derived from the authors' two FC72 JSONs — Stefan is linear in wall superheat:

- Twall-103 → 0.5873
- Twall-107 → 0.6395

**Formula:  `Stefan = 0.01305 × (Twall − 58)`** (intercept ≈ FC-72 saturation
temperature, ~58 °C).

| Target Twall | Stefan |
|---|---|
| 110 | 0.6788 |
| 115 | 0.7440 |
| **120** | **0.8091** |
| 130 | 0.9396 |
| 150 | 1.2006 |

### numSites (estimate)

Hotter walls nucleate more sites. We have no authors' correlation, so these are
**reasonable estimates**; exact positions are auto-placed by `flashInput.py`
(a Halton sequence), not from any JSON.

| Target Twall | numSites |
|---|---|
| 110 | 45 |
| 115 | 55 |
| **120** | **70** |
| 130 | 90 |
| 150 | 120 |

> **Honesty note:** Stefan values are faithful to the authors' FC72 trend. Site
> counts and positions are **not** from the authors — treat these as
> physically-plausible custom runs, not reproductions of a published dataset.

---

## PART A — Build everything (one-time setup)

### A1. Files you need on your Mac

In `/Users/sarah_yang/Downloads/`:
- `Outflow-Forcing-BubbleML-main/`  (this repo)
- `Flash-X-main/`  (the Flash-X source — can't be cloned, so we upload it)

### A2. Upload to Lambda (run on your **Mac**)

```bash
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/Outflow-Forcing-BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/

# Flash-X has internal symlinks, so scp -r fails on it directly.
# Tar it first. COPYFILE_DISABLE=1 stops macOS adding ._ junk files.
cd /Users/sarah_yang/Downloads
COPYFILE_DISABLE=1 tar czf Flash-X-main.tar.gz Flash-X-main
scp -i BubbleID.pem Flash-X-main.tar.gz ubuntu@132.145.142.183:/home/ubuntu/
```

### A3. Log in and extract Flash-X (run on **Lambda** from here on)

```bash
ssh -i BubbleID.pem ubuntu@132.145.142.183
cd ~
tar xzf Flash-X-main.tar.gz
```

### A4. Install system prerequisites

```bash
sudo apt update
sudo apt install -y build-essential gfortran openmpi-bin libopenmpi-dev
```

Build HDF5 from source (apt's version lacks parallel+Fortran together):

```bash
cd ~
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.2/src/hdf5-1.12.2.tar.gz
tar -xzf hdf5-1.12.2.tar.gz
cd hdf5-1.12.2
./configure --enable-parallel --enable-fortran CC=mpicc CXX=mpicxx FC=mpif90 --prefix=$HOME/hdf5-install
make -j$(nproc)
make install
```

Put HDF5 on your PATH permanently:

```bash
echo 'export PATH="$HOME/hdf5-install/bin:$PATH"' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH="$HOME/hdf5-install/lib:$LD_LIBRARY_PATH"' >> ~/.bashrc
source ~/.bashrc
```

Install Jobrunner and Python deps:

```bash
pip3 install PyJobRunner==2023.8 toml numpy scipy h5py
```

Verify:

```bash
mpicc --version && h5pfc --version && jobrunner --help
```

### A5. Create your site directory

```bash
cd ~/Outflow-Forcing-BubbleML-main
mkdir -p sites/lambda
echo '# No module system on Lambda' > sites/lambda/modules.sh
cp sites/sedona/Makefile.h.FlashX sites/lambda/Makefile.h.FlashX
```

**Edit the Makefile** to add a gfortran-10+ compatibility flag.

Option 1 — nano:
```bash
nano sites/lambda/Makefile.h.FlashX
```
Find:
```
FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized
```
Change to (append `-fallow-argument-mismatch`):
```
FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized -fallow-argument-mismatch
```
Save (Ctrl+O, Enter), exit (Ctrl+X).

Option 2 — one-liner:
```bash
sed -i 's/FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized/FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized -fallow-argument-mismatch/' sites/lambda/Makefile.h.FlashX
```

> Why: newer gfortran rejects the MPI calls in `nameValueLL_bcast.F90` with
> "Type mismatch…". This flag downgrades those errors to warnings.

### A6. Edit `config.sh`

```bash
nano ~/Outflow-Forcing-BubbleML-main/config.sh
```
| Find | Change to |
|---|---|
| `SiteName="sedona"` | `SiteName="lambda"` |
| `AMReX_Enable=true` | `AMReX_Enable=false` |
| `FlashX_Enable=true` | `FlashX_Enable=false` |

One-liner alternative:
```bash
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's/SiteName="sedona"/SiteName="lambda"/; s/AMReX_Enable=true/AMReX_Enable=false/; s/FlashX_Enable=true/FlashX_Enable=false/' config.sh
```

### A7. Place Flash-X and clean macOS junk

```bash
cp -r ~/Flash-X-main ~/Outflow-Forcing-BubbleML-main/software/Flash-X
find ~/Outflow-Forcing-BubbleML-main -name '._*' -delete
find ~/Outflow-Forcing-BubbleML-main -name '.DS_Store' -delete
```

> Why the `find`: macOS `._*` files look like `.F90-mc` source to the Flash-X
> setup parser and crash it with a `UnicodeDecodeError`.

### A8. Switch the setup scripts from SSH to HTTPS clones

```bash
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's|git@github.com:|https://github.com/|g' software/setupAMReX.sh software/setupFlashKit.sh software/setupHDF5.sh
```

> Why: Lambda has no GitHub SSH key, so SSH-style clone URLs fail.

### A9. Build AMReX 23.11

```bash
cd ~/Outflow-Forcing-BubbleML-main/software
git clone https://github.com/AMReX-Codes/amrex.git --branch 23.11 AMReX

cd ~/Outflow-Forcing-BubbleML-main
export PROJECT_HOME=$PWD
export MPI_HOME=$(dirname $(dirname $(which mpicc)))
export HDF5_HOME=$HOME/hdf5-install
export AMREX2D_HOME="$PWD/software/AMReX/install-lambda/2D"
export AMREX3D_HOME="$PWD/software/AMReX/install-lambda/3D"

cd software/AMReX
./configure --dim=2 --prefix=$AMREX2D_HOME
make -j$(nproc) && make install

make clean
./configure --dim=3 --prefix=$AMREX3D_HOME
make -j$(nproc) && make install
```

Verify:
```bash
ls $AMREX2D_HOME/lib/libamrex.a
```

> Why 23.11: the public Flash-X main branch needs it (references
> `amrex_interp_face_linear`, absent in the repo's pinned AMReX commit).

### A10. Edit the root `Jobfile` (scheduler)

```bash
sed -i 's/command: sbatch/command: bash/' ~/Outflow-Forcing-BubbleML-main/Jobfile
```

> Why: Lambda has no SLURM scheduler.

### A11. Install FlashKit

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner setup software
```

With AMReX/Flash-X/HDF5 disabled in `config.sh`, this only installs FlashKit.
**Part A done — the software stack is built.**

---

## PART B — Prepare the shared WallSuperheat machinery (FC72 world)

These edits are **study-wide** (shared by every Twall run). Do them once.

```bash
cd ~/Outflow-Forcing-BubbleML-main
SIM=simulation/PoolBoiling/WallSuperheat-FC72-2D
```

### B1. Fix the shared scripts

```bash
# flashOptions.sh — corrected flags, single line
echo 'FlashOptions="incompFlow/PoolBoiling -auto -maxblocks=100 +amrex +parallelIO -site=$SiteHome -makefile=FlashX InsForceInOut=True InsLSDamping=False InsExtras=True IOWriteGridFiles=True -2d -nxb=16 -nyb=16"' > $SIM/flashOptions.sh

# flashBuild.sh — drop the inaccessible git checkout
sed -i 's/cd $FLASHX_HOME && git checkout ef1d9026 && .\/setup/cd $FLASHX_HOME \&\& .\/setup/' $SIM/flashBuild.sh

# flashRestart.sh — neutralize (we start fresh; the checkpoint isn't in the repo)
echo '# fresh start - no checkpoint to restore' > $SIM/flashRestart.sh

# flashRun.sh — set MPI ranks. 1024 blocks; -np 16 = 64 blocks/rank (fits a 30-core box)
sed -i 's/mpirun job.target/mpirun -np 16 job.target/' $SIM/flashRun.sh
grep mpirun $SIM/flashRun.sh    # confirm: mpirun -np 16 job.target
```

> - `SimForceInOut`→`InsForceInOut`; add `InsLSDamping=False` + `InsExtras=True`
>   (defines `omgm`); remove the `\` jobrunner mis-parses.
> - `git checkout ef1d9026` fails (commit inaccessible, not a git repo).
> - The study ships as a restart run; the original `flashRestart.sh` copies a
>   missing checkpoint and aborts.
> - Bare `mpirun` grabs all cores → segfault. Pick a divisor of 1024 that is
>   ≤ your core count: `nproc` to check; use `-np 32` if you have ≥32 cores,
>   else `-np 16`. (1024 only has power-of-2 divisors.)

### B2. Set the shared parent `flash.toml` to FC72 properties

The parent holds domain, grid, boundaries (already correct: −8→8 × 0→16, 32×32,
noslip/outflow) plus fluid properties and heater geometry, which we set to the
authors' FC72 values.

Option 1 — nano (`nano $SIM/flash.toml`), make these changes:

| Find | Change to |
|---|---|
| `Heater.advAngle = 90.0` | `Heater.advAngle = 45.0` |
| `Heater.nucWaitTime = 0.2` | `Heater.nucWaitTime = 0.4` |
| `Heater.xmin = -5.0` | `Heater.xmin = -5.25` |
| `Heater.xmax = 5.0` | `Heater.xmax = 5.25` |
| `Parfile.ins_invReynolds = 0.0042` | `Parfile.ins_invReynolds = 0.0043` |
| `Parfile.ht_Prandtl = 8.4` | `Parfile.ht_Prandtl = 7.35` |
| `Parfile.mph_rhoGas = 0.0083` | `Parfile.mph_rhoGas = 0.008687` |
| `Parfile.mph_muGas = 1.0` | `Parfile.mph_muGas = 0.02816` |
| `Parfile.mph_thcoGas = 0.25` | `Parfile.mph_thcoGas = 0.209` |
| `Parfile.mph_CpGas = 0.83` | `Parfile.mph_CpGas = 0.7997` |

Option 2 — one-liner:
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
grep -E 'muGas|rhoGas|thcoGas|CpGas|Prandtl|invReynolds|advAngle|nucWaitTime' $SIM/flash.toml
```

> Leave `Heater.wallTemp = 1.0` — the sim runs in normalized temperature [0–1];
> the physical wall temp is encoded in `mph_Stefan` (set per-run in Part C).

---

## PART C — Create and run your custom hotter run

### C1. Set your target

```bash
cd ~/Outflow-Forcing-BubbleML-main
SIM=simulation/PoolBoiling/WallSuperheat-FC72-2D
TWALL=Twall-120        # <-- your target temperature
STEFAN=0.8091          # <-- from the Stefan table
NSITES=70              # <-- from the numSites table
```

> Re-run this block (and `SIM=`) if you open a new SSH session.

### C2. Create the run directory from an existing one

We copy the repo's `Twall-100` folder just to get its `Jobfile`/`flash.toml`
skeleton, then override the values.

```bash
cp -r $SIM/Twall-100 $SIM/$TWALL
rm -f $SIM/$TWALL/*hdf5* $SIM/$TWALL/flashx $SIM/$TWALL/flash.par \
      $SIM/$TWALL/merged.toml $SIM/$TWALL/INS_* $SIM/$TWALL/*.log 2>/dev/null
```

### C3. Set the per-run values (Stefan, sites, fresh start)

Option 1 — nano (`nano $SIM/$TWALL/flash.toml`):

| Find | Change to |
|---|---|
| `Heater.numSites = 40` | `Heater.numSites = 70` (your NSITES) |
| `Parfile.mph_Stefan = 0.5579` | `Parfile.mph_Stefan = 0.8091` (your STEFAN) |
| `Parfile.restart = ".true."` | `Parfile.restart = ".false."` |
| `Parfile.checkpointFileNumber = 20` | `Parfile.checkpointFileNumber = 0` |
| `Parfile.plotFileNumber = 100` | `Parfile.plotFileNumber = 0` |
| `Parfile.plotFileIntervalTime = 1.0` | `Parfile.plotFileIntervalTime = 0.1` |
| `Parfile.tmax = 200.0` | `Parfile.tmax = 20.0` (short test; raise later) |

Option 2 — one-liner (uses Part C1 variables):
```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
sed -i \
 -e "s/^Heater.numSites = .*/Heater.numSites = $NSITES/" \
 -e "s/^Parfile.mph_Stefan = .*/Parfile.mph_Stefan = $STEFAN/" \
 -e 's/^Parfile.restart = ".true."/Parfile.restart = ".false."/' \
 -e 's/^Parfile.checkpointFileNumber = .*/Parfile.checkpointFileNumber = 0/' \
 -e 's/^Parfile.plotFileNumber = .*/Parfile.plotFileNumber = 0/' \
 -e 's/^Parfile.plotFileIntervalTime = .*/Parfile.plotFileIntervalTime = 0.1/' \
 -e 's/^Parfile.tmax = .*/Parfile.tmax = 20.0/' \
 flash.toml
grep -E 'numSites|Stefan|restart|checkpointFileNumber|plotFileNumber|tmax' flash.toml
cd ~/Outflow-Forcing-BubbleML-main
```

### C4. Build

```bash
cd ~/Outflow-Forcing-BubbleML-main
rm -rf software/Flash-X/object
jobrunner setup $SIM/$TWALL
```

Expect `success`.

### C5. Generate inputs (merged) and fix the heater filename

The config is split across parent + subdir, so merge them for `flashInput.py`
(running it on the subdir alone fails with `KeyError: 'name'`):

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
( cat ../flash.toml; echo; cat flash.toml ) > merged.toml
python3 ../../../flashInput.py --input merged.toml
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001
cd ~/Outflow-Forcing-BubbleML-main
```

It should report writing the heater file with your `numSites` value
(Halton-placed sites, radius 0.2).

### C6. Run

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

> The `UCX WARN … incompatible` and `ignoring unknown parameter` messages are
> harmless noise on a single-node run. This is a 512×512 grid with many bubbles —
> expect it to be slow; the `tmax=20` test confirms it nucleates before you
> commit to a long `tmax=200` run.

---

## PART D — Convert to a BubbleML HDF5 file

### D1. Upload BubbleML (run on your **Mac**)

```bash
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/
```

### D2. Install conversion deps (on **Lambda**)

```bash
pip3 install boxkit==2023.6.7 torch h5py joblib matplotlib
```

### D3. Create the conversion script

Must live in BubbleML's `scripts/` so the import resolves.

```bash
nano ~/BubbleML-main/scripts/convert.py
```
Paste, then save and exit:
```python
from boxkit_dataset import BoilingDataset
import h5py

sim_dir = "/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-120"
output_file = "/home/ubuntu/PoolBoiling-Twall-120.hdf5"

print(f"Reading simulation data from: {sim_dir}")
b = BoilingDataset(sim_dir)

perm = (2, 0, 1)
with h5py.File(output_file, 'w') as f:
    f.create_dataset('temperature', data=b._data['temp'].permute(perm))
    f.create_dataset('velx',        data=b._data['velx'].permute(perm))
    f.create_dataset('vely',        data=b._data['vely'].permute(perm))
    f.create_dataset('dfun',        data=b._data['dfun'].permute(perm))
    f.create_dataset('pressure',    data=b._data['pres'].permute(perm))
    f.create_dataset('x',           data=b._data['x'].permute(perm))
    f.create_dataset('y',           data=b._data['y'].permute(perm))
    with h5py.File(b._filenames[0], 'r') as src:
        if 'real runtime parameters' in src:
            f.create_dataset('real-runtime-params', data=src['real runtime parameters'][:])
        if 'integer runtime parameters' in src:
            f.create_dataset('int-runtime-params', data=src['integer runtime parameters'][:])

print(f"Done! Written to {output_file}")
```

> This study outputs `pres, velx, vely, dfun, temp, dust` — not `mflx/nrmx/nrmy`,
> so only 7 fields are written (plus the two runtime-param sets). Including the
> missing vars would `KeyError`.

### D4. Run, verify, download

```bash
cd ~/BubbleML-main/scripts
python3 convert.py

python3 -c "
import h5py
with h5py.File('/home/ubuntu/PoolBoiling-Twall-120.hdf5','r') as f:
    print('Datasets:', list(f.keys()))
    print('temperature shape (T,Y,X):', f['temperature'].shape)
"
```

On your **Mac**:
```bash
scp -i BubbleID.pem ubuntu@132.145.142.183:/home/ubuntu/PoolBoiling-Twall-120.hdf5 ~/Downloads/
```

> Re-dimensionalize temperature for cross-temperature comparison: multiply the
> normalized [0–1] `temperature` field by the wall temp (e.g. 120).

---

## Making another temperature

Parts A and B are one-time. For each new temperature, just redo Part C (and D)
with new values — e.g. Twall-150:

```bash
TWALL=Twall-150 ; STEFAN=1.2006 ; NSITES=120
```

Within the FC72 world that's genuinely all that changes: **Stefan + numSites.**

---

## All changes vs the stock repo (quick reference)

| File / action | Change | Reason |
|---|---|---|
| Flash-X upload | `tar` with `COPYFILE_DISABLE=1` | symlinks + macOS `._*` files |
| `sites/lambda/modules.sh` | one-line stub | no `module` system |
| `sites/lambda/Makefile.h.FlashX` | add `-fallow-argument-mismatch` | gfortran 10+ MPI type checks |
| `config.sh` | `SiteName=lambda`, disable AMReX/FlashX | manual installs |
| `software/setup*.sh` | `git@github.com:` → `https://` | no SSH keys |
| AMReX | build **23.11** | match Flash-X main |
| `Jobfile` | `sbatch` → `bash` | no SLURM |
| `flashOptions.sh` | `Ins*` flags, `+InsExtras=True`, one line | renames, `omgm_var`, parse bug |
| `flashBuild.sh` | drop `git checkout ef1d9026` | commit inaccessible / not a git repo |
| `flashRestart.sh` | emptied to a no-op | missing checkpoint aborts submit |
| `flashRun.sh` | `mpirun -np 16` (or 32) | bare mpirun segfaults; divisor of 1024 ≤ cores |
| parent `flash.toml` | FC72 props, Re, Pr, heater angles/extent | FC72 world |
| `Twall-XXX/flash.toml` | custom Stefan + numSites, fresh start, dt 0.1 | "hotter" custom run |
| heater file | copy to `flash_hdf5_htr_0001` | filename mismatch |
| `convert.py` | in `scripts/`, params from source, 7 fields | import path; empty params; var set |
