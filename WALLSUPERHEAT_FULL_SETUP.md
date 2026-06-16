# Running PoolBoiling-Saturated-FC72-2D (Twall-107) on a Lambda GPU Instance

A **fully self-contained** guide: from a fresh Lambda instance to a running
pool-boiling simulation that matches the published
**PoolBoiling-Saturated-FC72-2D** dataset at wall temperature **107**, then a
BubbleML HDF5 file. Nothing is assumed to be pre-built. Every file edit is shown
explicitly.

This uses the repo's `WallSuperheat-FC72-2D` machinery (its domain, grid, and
boundaries already match the published run) and updates the **fluid properties,
Stefan number, heater settings, and nucleation sites** to the authors' values
from `Twall_107.json`.

- Instance IP used below: `132.145.142.183` — replace with yours.
- SSH key: `BubbleID.pem` (in your Mac's current folder).

---

## How to edit files on Lambda (quick primer)

Most edits use the **nano** editor:

1. Open: `nano <path-to-file>`
2. Move with **arrow keys**, edit by typing.
3. Save: **Ctrl+O**, then **Enter**.
4. Exit: **Ctrl+X**.

Where faster, a one-line `sed` alternative is given that makes the same change
without an editor. Use whichever you prefer — **not both**.

---

## PART A — Build everything (one-time setup)

### A1. Files you need on your Mac

In `/Users/sarah_yang/Downloads/` you should have:
- `Outflow-Forcing-BubbleML-main/`  (this repo)
- `Flash-X-main/`  (the Flash-X source — can't be cloned, so we upload it)
- `Twall_107.json`  (the authors' parameters — used in Part B)

### A2. Upload to Lambda (run on your **Mac**)

```bash
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/Outflow-Forcing-BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/
scp -i BubbleID.pem /Users/sarah_yang/Downloads/Twall_107.json ubuntu@132.145.142.183:/home/ubuntu/

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

Build HDF5 from source (the apt version lacks parallel+Fortran together):

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

Verify everything is found:

```bash
mpicc --version && h5pfc --version && jobrunner --help
```

### A5. Create your site directory

```bash
cd ~/Outflow-Forcing-BubbleML-main
mkdir -p sites/lambda
```

Create an (essentially empty) modules file — Lambda has no `module` system:

```bash
echo '# No module system on Lambda' > sites/lambda/modules.sh
```

Copy the Makefile template from the `sedona` site:

```bash
cp sites/sedona/Makefile.h.FlashX sites/lambda/Makefile.h.FlashX
```

**Edit the Makefile** to add a gfortran-10+ compatibility flag.

Option 1 — with nano:
```bash
nano sites/lambda/Makefile.h.FlashX
```
Find this line:
```
FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized
```
Change it to (append `-fallow-argument-mismatch`):
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
Make these three changes:

| Find | Change to |
|---|---|
| `SiteName="sedona"` | `SiteName="lambda"` |
| `AMReX_Enable=true` | `AMReX_Enable=false` |
| `FlashX_Enable=true` | `FlashX_Enable=false` |

Save and exit.

One-liner alternative:
```bash
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's/SiteName="sedona"/SiteName="lambda"/; s/AMReX_Enable=true/AMReX_Enable=false/; s/FlashX_Enable=true/FlashX_Enable=false/' config.sh
```

> Why: point at our site; disable AMReX/Flash-X cloning (Flash-X is uploaded;
> AMReX we build next).

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
nano ~/Outflow-Forcing-BubbleML-main/software/setupFlashKit.sh
```
Find:
```
git clone git@github.com:akashdhruv/FlashKit --branch main FlashKit && cd FlashKit
```
Change `git@github.com:` to `https://github.com/`:
```
git clone https://github.com/akashdhruv/FlashKit --branch main FlashKit && cd FlashKit
```
Save and exit.

One-liner alternative (all three scripts at once):
```bash
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's|git@github.com:|https://github.com/|g' software/setupAMReX.sh software/setupFlashKit.sh software/setupHDF5.sh
```

> Why: Lambda has no GitHub SSH key, so SSH-style clone URLs fail.

### A9. Build AMReX 23.11

The public Flash-X main branch needs **AMReX 23.11** (not the commit pinned in
`config.sh`).

```bash
cd ~/Outflow-Forcing-BubbleML-main/software
git clone https://github.com/AMReX-Codes/amrex.git --branch 23.11 AMReX
```

Set up the build environment (needed for this shell session):

```bash
cd ~/Outflow-Forcing-BubbleML-main
export PROJECT_HOME=$PWD
export MPI_HOME=$(dirname $(dirname $(which mpicc)))
export HDF5_HOME=$HOME/hdf5-install
export AMREX2D_HOME="$PWD/software/AMReX/install-lambda/2D"
export AMREX3D_HOME="$PWD/software/AMReX/install-lambda/3D"
```

Build 2D, then 3D:

```bash
cd software/AMReX
./configure --dim=2 --prefix=$AMREX2D_HOME
make -j$(nproc) && make install

make clean
./configure --dim=3 --prefix=$AMREX3D_HOME
make -j$(nproc) && make install
```

Verify the 2D library exists:
```bash
ls $AMREX2D_HOME/lib/libamrex.a
```

### A10. Edit the root `Jobfile` (scheduler)

```bash
nano ~/Outflow-Forcing-BubbleML-main/Jobfile
```
Find `  command: sbatch` and change to `  command: bash`. Save and exit.

One-liner alternative:
```bash
sed -i 's/command: sbatch/command: bash/' ~/Outflow-Forcing-BubbleML-main/Jobfile
```

> Why: Lambda has no SLURM scheduler, so `sbatch` doesn't exist.

### A11. Install FlashKit

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner setup software
```

With AMReX/Flash-X/HDF5 disabled in `config.sh`, this only installs FlashKit.
**Part A is done — the software stack is built.**

---

## PART B — Configure and run PoolBoiling-Saturated Twall-107

The repo's `WallSuperheat-FC72-2D` directory already matches the published run's
**geometry** (domain `−8→8 × 0→16`, `32×32` blocks, noslip side walls, outflow
top). What differs are the **fluid properties, Stefan number, heater settings,
and nucleation sites**, summarized here:

| Parameter | Repo WallSuperheat | Author Twall-107 | Lives in |
|---|---|---|---|
| domain / blocks / boundaries | −8→8, 32×32, noslip | same | parent (no change) |
| `Heater.advAngle` | 90.0 | **45.0** | parent |
| `Heater.nucWaitTime` | 0.2 | **0.4** | parent |
| `Heater.xmin` / `xmax` | −5.0 / 5.0 | **−5.25 / 5.25** | parent |
| `Parfile.ins_invReynolds` | 0.0042 | **0.0043** | parent |
| `Parfile.ht_Prandtl` | 8.4 | **7.35** | parent |
| `Parfile.mph_rhoGas` | 0.0083 | **0.008687** | parent |
| `Parfile.mph_muGas` | 1.0 | **0.02816** | parent |
| `Parfile.mph_thcoGas` | 0.25 | **0.209** | parent |
| `Parfile.mph_CpGas` | 0.83 | **0.7997** | parent |
| `Heater.numSites` | 40 (Twall-100) | **37** | subdir |
| `Parfile.mph_Stefan` | 0.5579 (Twall-100) | **0.6395** | subdir |
| `Parfile.plotFileIntervalTime` | 1.0 | **0.1** | subdir |

All of these are **runtime** parameters → **no recompile needed** after editing.

### B1. Create a Twall-107 run directory

There's no `Twall-107/` in the repo, so make one from `Twall-100`:

```bash
cd ~/Outflow-Forcing-BubbleML-main
SIM=simulation/PoolBoiling/WallSuperheat-FC72-2D
TWALL=Twall-107
cp -r $SIM/Twall-100 $SIM/$TWALL
```

> `TWALL` and `SIM` only last for the current SSH session — re-run these three
> lines if you log out and back in.

### B2. Edit `flashOptions.sh` (parent)

```bash
nano $SIM/flashOptions.sh
```
Replace the entire contents with this **single line**:
```
FlashOptions="incompFlow/PoolBoiling -auto -maxblocks=100 +amrex +parallelIO -site=$SiteHome -makefile=FlashX InsForceInOut=True InsLSDamping=False InsExtras=True IOWriteGridFiles=True -2d -nxb=16 -nyb=16"
```
Save and exit.

One-liner alternative:
```bash
echo 'FlashOptions="incompFlow/PoolBoiling -auto -maxblocks=100 +amrex +parallelIO -site=$SiteHome -makefile=FlashX InsForceInOut=True InsLSDamping=False InsExtras=True IOWriteGridFiles=True -2d -nxb=16 -nyb=16"' > $SIM/flashOptions.sh
```

> Why: `SimForceInOut`→`InsForceInOut`; add `InsLSDamping=False` and
> `InsExtras=True` (defines `omgm`); remove the `\` line continuation jobrunner
> mis-parses.

### B3. Edit `flashBuild.sh` (parent)

```bash
nano $SIM/flashBuild.sh
```
Find:
```
cd $FLASHX_HOME && git checkout ef1d9026 && ./setup $FlashOptions
```
Change to:
```
cd $FLASHX_HOME && ./setup $FlashOptions
```
Save and exit.

One-liner alternative:
```bash
sed -i 's/cd $FLASHX_HOME && git checkout ef1d9026 && .\/setup/cd $FLASHX_HOME \&\& .\/setup/' $SIM/flashBuild.sh
```

> Why: that commit isn't accessible and our Flash-X isn't a git repo.

### B4. Neutralize `flashRestart.sh` (parent)

```bash
nano $SIM/flashRestart.sh
```
Delete everything, replace with one comment line:
```
# fresh start - no checkpoint to restore
```
Save and exit.

One-liner alternative:
```bash
echo '# fresh start - no checkpoint to restore' > $SIM/flashRestart.sh
```

> Why: this study ships as a *restart* run; the original copies a checkpoint
> file that isn't in the repo, aborting the submit. We start fresh.

### B5. Edit `flashRun.sh` (parent) — MPI process count

```bash
nano $SIM/flashRun.sh
```
In the `else` branch near the bottom, change `	mpirun job.target` to:
```
	mpirun -np 32 job.target
```
Save and exit.

One-liner alternative:
```bash
sed -i 's/^\tmpirun job.target/\tmpirun -np 32 job.target/' $SIM/flashRun.sh
grep mpirun $SIM/flashRun.sh    # confirm: mpirun -np 32 job.target
```

> Why: the grid is 32×32 = 1024 blocks; a bare `mpirun` grabs all 64 cores and
> some ranks get zero blocks → segfault. 32 ranks → 32 blocks each.

### B6. Edit the **parent** `flash.toml` — shared FC72 physics & heater

```bash
nano $SIM/flash.toml
```
Make these changes (see the table above):

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

Save and exit.

One-liner alternative:
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

> Leave `Heater.wallTemp = 1.0` — the sim runs in normalized temperature [0–1];
> the physical "107" is encoded in `mph_Stefan = 0.6395` (set next). Multiply by
> 107 at visualization time to get degrees.

### B7. Edit the **Twall-107** `flash.toml` — sites, Stefan, fresh start

```bash
nano $SIM/$TWALL/flash.toml
```
Make these changes:

| Find | Change to |
|---|---|
| `Heater.numSites = 40` | `Heater.numSites = 37` |
| `Parfile.mph_Stefan = 0.5579` | `Parfile.mph_Stefan = 0.6395` |
| `Parfile.restart = ".true."` | `Parfile.restart = ".false."` |
| `Parfile.checkpointFileNumber = 20` | `Parfile.checkpointFileNumber = 0` |
| `Parfile.plotFileNumber = 100` | `Parfile.plotFileNumber = 0` |
| `Parfile.plotFileIntervalTime = 1.0` | `Parfile.plotFileIntervalTime = 0.1` |
| `Parfile.tmax = 200.0` | `Parfile.tmax = 20.0` |

Save and exit. (`tmax=20` is a short test; raise to `200.0` for the full run.)

One-liner alternative:
```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
sed -i \
 -e 's/^Heater.numSites = .*/Heater.numSites = 37/' \
 -e 's/^Parfile.mph_Stefan = .*/Parfile.mph_Stefan = 0.6395/' \
 -e 's/^Parfile.restart = ".true."/Parfile.restart = ".false."/' \
 -e 's/^Parfile.checkpointFileNumber = .*/Parfile.checkpointFileNumber = 0/' \
 -e 's/^Parfile.plotFileNumber = .*/Parfile.plotFileNumber = 0/' \
 -e 's/^Parfile.plotFileIntervalTime = .*/Parfile.plotFileIntervalTime = 0.1/' \
 -e 's/^Parfile.tmax = .*/Parfile.tmax = 20.0/' \
 flash.toml
cd ~/Outflow-Forcing-BubbleML-main
```

### B8. Build the simulation

```bash
cd ~/Outflow-Forcing-BubbleML-main
rm -rf software/Flash-X/object
jobrunner setup $SIM/$TWALL
```

Expect `success`.

### B9. Generate inputs and fix the heater filename

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
#python3 ../../../flashInput.py --input flash.toml
( cat ../flash.toml; echo; cat flash.toml ) > merged.toml
python3 ../../../flashInput.py --input merged.toml

cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001
```

> `flashInput.py` is three levels up from a `Twall-XX` folder. The heater is
> named `pool_boiling`, but Flash-X looks for the generic `flash_hdf5_htr_0001`,
> so we copy it.

#### B9 (optional) — Inject the authors' EXACT nucleation sites

With `numSites=37`, `flashInput.py` places sites using a **Halton sequence** and
hardcodes seed radii to **0.2** — so positions and radii won't match the authors'
exact `Twall_107.json` (which uses 37 specific locations, radii 0.1). For a
faithful reproduction, overwrite the site arrays from the JSON:

```bash
cd ~/Outflow-Forcing-BubbleML-main/$SIM/$TWALL
python3 - <<'PYEOF'
import h5py, json, numpy as np
j = json.load(open('/home/ubuntu/Twall_107.json'))
xs = np.array(j['nuc_sites_x'], dtype='float32')
ys = np.array(j['nuc_sites_y'], dtype='float32')
rr = np.array(j['nuc_seed_radii'], dtype='float32')
n  = len(xs)
with h5py.File('pool_boiling_hdf5_htr_0001', 'r+') as h:
    for name in ['site/num','site/x','site/y','site/z','init/radii']:
        if name in h: del h[name]
    h.create_dataset('site/num',  data=np.array([n], dtype='int32'))
    h.create_dataset('site/x',    data=xs)
    h.create_dataset('site/y',    data=ys)
    h.create_dataset('site/z',    data=np.zeros(n, dtype='float32'))
    h.create_dataset('init/radii',data=rr)
print(f'Injected {n} exact nucleation sites')
PYEOF
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001   # re-copy after editing
```

> Skip this block if an approximate 37-site layout is good enough.

### B10. Run

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner submit $SIM/$TWALL
```

`Submitting job` with no further output means it's running.

### B11. Monitor (second SSH session)

```bash
ssh -i BubbleID.pem ubuntu@132.145.142.183
```
```bash
DIR=~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-107
tail -f $DIR/INS_Pool_Boiling.log
ls $DIR/*plt_cnt* 2>/dev/null | wc -l
ps aux | grep flashx
```

> The `UCX WARN … incompatible` and `ignoring unknown parameter` messages are
> harmless noise on a single-node run.

---

## PART C — Convert the output to a BubbleML HDF5 file

### C1. Upload BubbleML (run on your **Mac**)

```bash
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/
```

### C2. Install conversion deps (on **Lambda**)

```bash
pip3 install boxkit==2023.6.7 torch h5py joblib matplotlib
```

### C3. Create the conversion script

It must live inside BubbleML's `scripts/` folder so the import resolves.

```bash
nano ~/BubbleML-main/scripts/convert.py
```
Paste this, then save and exit:
```python
from boxkit_dataset import BoilingDataset
import h5py

sim_dir = "/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/WallSuperheat-FC72-2D/Twall-107"
output_file = "/home/ubuntu/PoolBoiling-Saturated-Twall-107.hdf5"

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

> This study's plot variables are `pres, velx, vely, dfun, temp, dust` — it does
> **not** output `mflx`, `nrmx`, `nrmy`, so those datasets are omitted here
> (including them would `KeyError`).

### C4. Run the conversion

```bash
cd ~/BubbleML-main/scripts
python3 convert.py
```

### C5. Verify

```bash
python3 -c "
import h5py
with h5py.File('/home/ubuntu/PoolBoiling-Saturated-Twall-107.hdf5','r') as f:
    print('Datasets:', list(f.keys()))
    print('temperature shape (T,Y,X):', f['temperature'].shape)
"
```

The frame count `T` equals `(number of plot files − 1)` — the converter drops
the final plot file by design.

### C6. Download to your Mac (run on your **Mac**)

```bash
scp -i BubbleID.pem ubuntu@132.145.142.183:/home/ubuntu/PoolBoiling-Saturated-Twall-107.hdf5 ~/Downloads/
```

> **Re-dimensionalize temperature** before comparing across wall temperatures:
> multiply the normalized [0–1] `temperature` field by 107.

---

## All changes vs the stock repo (quick reference)

| File / action | Change | Reason |
|---|---|---|
| Flash-X upload | `tar` with `COPYFILE_DISABLE=1` | symlinks + macOS `._*` files |
| `sites/lambda/modules.sh` | one-line stub | no `module` system |
| `sites/lambda/Makefile.h.FlashX` | add `-fallow-argument-mismatch` | gfortran 10+ MPI type checks |
| `config.sh` | `SiteName=lambda`, disable AMReX/FlashX | manual installs |
| `software/setup*.sh` | `git@github.com:` → `https://` | no SSH keys |
| AMReX | build **23.11** not the pinned commit | match Flash-X main |
| `Jobfile` | `sbatch` → `bash` | no SLURM |
| `flashOptions.sh` | `Ins*` flags, `+InsExtras=True`, one line | renames, `omgm_var`, parse bug |
| `flashBuild.sh` | drop `git checkout ef1d9026` | commit inaccessible / not a git repo |
| `flashRestart.sh` | emptied to a no-op | missing checkpoint aborts submit |
| `flashRun.sh` | `mpirun -np 32 job.target` | bare mpirun segfaults; 32 divides 1024 |
| parent `flash.toml` | FC72 props, Re, Pr, heater angles/extent | match published Twall-107 |
| `Twall-107/flash.toml` | 37 sites, Stefan 0.6395, fresh start, dt 0.1 | match published Twall-107 |
| heater file | exact-site injection + copy to `flash_hdf5_htr_0001` | faithful sites + filename fix |
| `convert.py` | in `scripts/`, params from source, 7 fields | import path; empty params; var set |
