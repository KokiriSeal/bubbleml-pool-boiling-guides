# Running BubbleML Simulations on a Lambda GPU Instance

End-to-end guide: build Flash-X, run a Pool Boiling simulation (matching the
published **SingleBubble-Saturated-FC72-2D** dataset), convert the output to a
BubbleML HDF5 dataset, and visualize it. Every step includes the fixes
discovered while getting this working on Lambda (Ubuntu 22.04).

Instance IP used below: `132.145.142.183` — replace with your own.
SSH key: `BubbleID.pem`.

> **Important context:** the repo's `simulation/PoolBoiling/SingleBubble/`
> directory is a small **demo** config — it is NOT the configuration that
> produced the published BubbleML SingleBubble dataset. Phase 12 below rewrites
> its `flash.toml` to match the real published parameters. If you only want a
> quick smoke-test, you can skip Phase 12 and run the demo as-is, but the result
> will look different from the dataset (see the note in Phase 12).

---

## Phase 1 — Upload files to Lambda

Run on your **Mac**.

```bash
# Upload the reproducibility repo
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/Outflow-Forcing-BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/

# Flash-X contains symlinks, so scp -r fails. Tar it first.
# COPYFILE_DISABLE=1 stops macOS from adding ._ resource-fork files.
cd /Users/sarah_yang/Downloads
COPYFILE_DISABLE=1 tar czf Flash-X-main.tar.gz Flash-X-main
scp -i BubbleID.pem Flash-X-main.tar.gz ubuntu@132.145.142.183:/home/ubuntu/
```

SSH in and extract:

```bash
ssh -i BubbleID.pem ubuntu@132.145.142.183
cd ~
tar xzf Flash-X-main.tar.gz
```

> **Bug fixed:** `scp -r` on the raw Flash-X folder fails silently because of
> internal symlinks. Tarring it first avoids that. `COPYFILE_DISABLE=1` prevents
> the macOS `._*` files that later crash the Flash-X setup script with a
> `UnicodeDecodeError`.

---

## Phase 2 — Install prerequisites

Run on **Lambda**.

```bash
sudo apt update
sudo apt install -y build-essential gfortran openmpi-bin libopenmpi-dev

# Build HDF5 from source with parallel + Fortran support
cd ~
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.2/src/hdf5-1.12.2.tar.gz
tar -xzf hdf5-1.12.2.tar.gz
cd hdf5-1.12.2
./configure --enable-parallel --enable-fortran CC=mpicc CXX=mpicxx FC=mpif90 --prefix=$HOME/hdf5-install
make -j$(nproc)
make install

# Put HDF5 on PATH permanently
echo 'export PATH="$HOME/hdf5-install/bin:$PATH"' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH="$HOME/hdf5-install/lib:$LD_LIBRARY_PATH"' >> ~/.bashrc
source ~/.bashrc

# Jobrunner + Python deps
pip3 install PyJobRunner==2023.8 toml numpy scipy h5py
```

Verify:

```bash
mpicc --version && h5pfc --version && jobrunner --help
```

---

## Phase 3 — Create your site directory

```bash
cd ~/Outflow-Forcing-BubbleML-main
mkdir -p sites/lambda

# modules.sh — Lambda has no `module` system, so make it a harmless one-liner
echo '# No module system on Lambda' > sites/lambda/modules.sh

# Reuse sedona's Makefile (it already targets GCC/Ubuntu)
cp sites/sedona/Makefile.h.FlashX sites/lambda/Makefile.h.FlashX

# Add -fallow-argument-mismatch for gfortran 10+ (MPI type-check errors)
sed -i 's/FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized/FFLAGS_OPT   = -c -O2 -fdefault-real-8 -fdefault-double-8 -Wuninitialized -fallow-argument-mismatch/' sites/lambda/Makefile.h.FlashX
```

> **Bug fixed (modules.sh):** an earlier heredoc left a literal `EOF` line in the
> file, which ran as a command. A single `echo` avoids that.
>
> **Bug fixed (gfortran 10+):** newer gfortran rejects the MPI calls in
> `nameValueLL_bcast.F90` with "Type mismatch between actual argument…".
> `-fallow-argument-mismatch` downgrades these to warnings.

---

## Phase 4 — Configure `config.sh`

```bash
cd ~/Outflow-Forcing-BubbleML-main

sed -i 's/SiteName="sedona"/SiteName="lambda"/' config.sh   # use our site
sed -i 's/FlashX_Enable=true/FlashX_Enable=false/' config.sh # we placed Flash-X manually
sed -i 's/AMReX_Enable=true/AMReX_Enable=false/' config.sh   # we build AMReX manually
```

---

## Phase 5 — Place Flash-X and fix the setup scripts

```bash
# Put Flash-X where the build expects it
cp -r ~/Flash-X-main ~/Outflow-Forcing-BubbleML-main/software/Flash-X

# Remove any macOS junk that crashes the Flash-X setup parser
find ~/Outflow-Forcing-BubbleML-main -name '._*' -delete
find ~/Outflow-Forcing-BubbleML-main -name '.DS_Store' -delete

# Switch clone URLs from SSH to HTTPS (no GitHub SSH keys on Lambda)
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's|git@github.com:|https://github.com/|g' software/setupAMReX.sh
sed -i 's|git@github.com:|https://github.com/|g' software/setupFlashKit.sh
sed -i 's|git@github.com:|https://github.com/|g' software/setupHDF5.sh
```

> **Bug fixed (UnicodeDecodeError):** macOS `._*` resource-fork files look like
> `.F90-mc` source files to the Flash-X macro processor, which then chokes on
> their binary bytes. Deleting them resolves it.

---

## Phase 6 — Build AMReX 23.11

The repo pins AMReX commit `cff96a9`, but the public **Flash-X main** branch
needs **AMReX 23.11** (it references `amrex_interp_face_linear`, absent in the
old commit). Build 23.11 instead.

```bash
cd ~/Outflow-Forcing-BubbleML-main/software
git clone https://github.com/AMReX-Codes/amrex.git --branch 23.11 AMReX

# Set up environment for the build
cd ~/Outflow-Forcing-BubbleML-main
source config.sh
source sites/lambda/modules.sh
export MPI_HOME=$(dirname $(dirname $(which mpicc)))
export HDF5_HOME=$HOME/hdf5-install
export AMREX2D_HOME="$PWD/software/AMReX/install-lambda/2D"
export AMREX3D_HOME="$PWD/software/AMReX/install-lambda/3D"

# 2D build
cd software/AMReX
./configure --dim=2 --prefix=$AMREX2D_HOME
make -j$(nproc) && make install

# 3D build
make clean
./configure --dim=3 --prefix=$AMREX3D_HOME
make -j$(nproc) && make install
```

> **Bug fixed (`amrex_interp_face_linear` not found):** Flash-X main and AMReX
> `cff96a9` are incompatible. AMReX 23.11 is the version the Flash-X README
> requires.

---

## Phase 7 — Fix the simulation's `flashOptions.sh`

The setup flag names changed in newer Flash-X, and jobrunner mis-parses the
backslash line continuation. Write it as one line.

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
echo 'FlashOptions="incompFlow/PoolBoiling -auto -maxblocks=100 +amrex +parallelIO -site=$SiteHome -makefile=FlashX InsForceInOut=True InsLSDamping=False InsExtras=True IOWriteGridFiles=True -2d -nxb=16 -nyb=16"' > flashOptions.sh
```

> **Bugs fixed:**
> - `SimForceInOut` → `InsForceInOut`, `SimLSDamping` → `InsLSDamping` (renamed).
> - Added `InsExtras=True` — defines the `omgm` vorticity variable; without it the
>   build fails with "Symbol 'omgm_var' has no IMPLICIT type".
> - Removed the trailing `\` line continuation — jobrunner treated it as a stray
>   argument ("Multiple problem names given. winner=\").

---

## Phase 8 — Fix `flashBuild.sh`

Your Flash-X is an unzipped tarball, not a git repo, so the `git checkout` fails.

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
sed -i 's/cd $FLASHX_HOME && git checkout afee52b0 && .\/setup/cd $FLASHX_HOME \&\& .\/setup/' flashBuild.sh
```

> **Bug fixed:** `git checkout afee52b0` errored with "not a git repository". We
> can't check out that pinned commit (the private Flash-X repo isn't accessible),
> so we drop the checkout and use the main branch we have.

---

## Phase 9 — Fix the scheduler in the root `Jobfile`

Lambda has no SLURM, so replace `sbatch` with `bash`.

```bash
cd ~/Outflow-Forcing-BubbleML-main
sed -i 's/command: sbatch/command: bash/' Jobfile
```

> **Bug fixed:** `jobrunner submit` failed with `sbatch: not found` (exit 127).

---

## Phase 10 — `jobrunner setup software`

With AMReX/Flash-X/HDF5 all disabled in `config.sh`, this only installs FlashKit.

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner setup software
```

---

## Phase 11 — `jobrunner setup simulation/PoolBoiling/SingleBubble`

Runs Flash-X `./setup`, compiles, and copies the `flashx` binary into the sim
directory.

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner setup simulation/PoolBoiling/SingleBubble
```

Expect to see `success`. If you change `flashOptions.sh` later, delete
`software/Flash-X/object` and re-run this step.

---

## Phase 12 — Match the published SingleBubble-Saturated-FC72-2D dataset

The repo's `SingleBubble/flash.toml` is a **demo**: a narrow half-domain
(`x: 0→2`) with a **slip (symmetry) wall** on the left and the single bubble
nucleating at `x = 0` — i.e. against that wall. The result is a half-bubble with
its thermal plume hugging the **bottom-left corner**, on generic (non-FC72)
fluid properties.

The published dataset (parameters from the authors' `Twall_103.json`) instead
uses a **centered full domain** (`x: −3→3`) with **noslip walls** on both sides,
the bubble at the **center**, and real **FC72** properties. That's why a stock
demo run looks different from the dataset (off-center plume, different physics).

These are all **runtime** parameters, so **no recompile is needed** — just edit
`flash.toml`, regenerate inputs (Phase 13), and run.

| Parameter | Demo (repo) | Published Twall-103 |
|---|---|---|
| `Parfile.xmin` / `xmax` | 0 / 2 | **−3 / 3** (centered) |
| `Parfile.ymax` | 6 | **9** |
| `Parfile.xl_boundary_type` | slip_ins | **noslip_ins** (full bubble) |
| `Parfile.nblockx` | 6 | **12** (→192 wide) |
| `Parfile.ins_invReynolds` | 0.0033 | **0.0043** |
| `Parfile.ht_Prandtl` | 7 | **7.35** |
| `Parfile.mph_Stefan` | 0.16 | **0.5873** (encodes Twall=103) |
| `Parfile.mph_rhoGas` | 0.0049 | **0.008687** |
| `Parfile.mph_muGas` | 0.02 | **0.02816** |
| `Parfile.mph_thcoGas` | 0.143 | **0.209** |
| `Parfile.mph_CpGas` | 0.735 | **0.7997** |
| `Heater.xmin` / `xmax` | −2 / 2 | **−3 / 3** |
| `Heater.nucSeedRadius` | 0.15 | **0.2** |
| `Parfile.tmax` | 250 | **400** |
| `Parfile.plotFileIntervalTime` | 0.5 | **0.2** |

Apply them all:

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
sed -i \
 -e 's/^Parfile.xmin = .*/Parfile.xmin = -3.0/' \
 -e 's/^Parfile.xmax = .*/Parfile.xmax = 3.0/' \
 -e 's/^Parfile.ymax = .*/Parfile.ymax = 9.0/' \
 -e 's/^Parfile.xl_boundary_type = .*/Parfile.xl_boundary_type = "noslip_ins"/' \
 -e 's/^Parfile.nblockx = .*/Parfile.nblockx = 12/' \
 -e 's/^Parfile.ins_invReynolds = .*/Parfile.ins_invReynolds = 0.0043/' \
 -e 's/^Parfile.ht_Prandtl = .*/Parfile.ht_Prandtl = 7.35/' \
 -e 's/^Parfile.mph_Stefan = .*/Parfile.mph_Stefan = 0.5873/' \
 -e 's/^Parfile.mph_rhoGas = .*/Parfile.mph_rhoGas = 0.008687/' \
 -e 's/^Parfile.mph_muGas = .*/Parfile.mph_muGas = 0.02816/' \
 -e 's/^Parfile.mph_thcoGas = .*/Parfile.mph_thcoGas = 0.209/' \
 -e 's/^Parfile.mph_CpGas = .*/Parfile.mph_CpGas = 0.7997/' \
 -e 's/^Parfile.tmax = .*/Parfile.tmax = 400/' \
 -e 's/^Parfile.plotFileIntervalTime = .*/Parfile.plotFileIntervalTime = 0.2/' \
 -e 's/^Heater.xmin = .*/Heater.xmin = -3.0/' \
 -e 's/^Heater.xmax = .*/Heater.xmax = 3.0/' \
 -e 's/^Heater.nucSeedRadius = .*/Heater.nucSeedRadius = 0.2/' \
 flash.toml

# verify
grep -E 'xmin|xmax|ymax|xl_boundary|nblockx|invReynolds|Prandtl|Stefan|rhoGas|muGas|thcoGas|CpGas|tmax|nucSeedRadius' flash.toml
```

> **Notes:**
> - **Leave `Heater.wallTemp = 1.0`.** The simulation runs in normalized
>   temperature [0–1]; the physical "103" is encoded in `mph_Stefan = 0.5873`.
>   When visualizing, multiply the normalized field by 103 to get degrees.
> - `tmax=400` with output every 0.2 = ~2000 frames — that's the full published
>   run and is **slow** on CPU. For a first test set `tmax` to ~20, confirm the
>   plume is now **centered**, then scale up.
> - For other wall temperatures, the main change is `mph_Stefan` (each `Twall`
>   maps to a different Stefan number in the saturated study).
> - **To keep the demo instead:** skip this phase. The demo still runs; it just
>   produces a corner-plume half-bubble on generic properties.

---

## Phase 13 — Generate input files (and fix the heater filename)

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble

# Generates flash.par and pool_boiling_hdf5_htr_0001 from flash.toml
python3 ../../flashInput.py --input flash.toml

# Flash-X looks for a generically named heater file — provide it
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001
```

> **Bug fixed:** the run aborted with HDF5 "unable to open file
> 'flash_hdf5_htr_0001'". The generator writes `pool_boiling_hdf5_htr_0001`;
> copying it to the expected name fixes the mismatch.
>
> Re-run this phase any time you edit `flash.toml`.

---

## Phase 14 — Set the MPI process count

The process count must divide the block count and give each rank ≤ 100 blocks.
- Demo grid: 6×18 = **108** blocks.
- Published grid (Phase 12): 12×18 = **216** blocks.

`-np 12` works for both (108÷12 = 9, 216÷12 = 18 blocks/rank).

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
sed -i 's|mpirun $JobWorkDir/job.target|mpirun -np 12 $JobWorkDir/job.target|' flashRun.sh
```

> **Bug fixed:** bare `mpirun` launched 64 ranks → "signal 11 (Segmentation
> fault)". Limiting to a divisor of the block count resolves it.

---

## Phase 15 — Run the simulation

```bash
cd ~/Outflow-Forcing-BubbleML-main
jobrunner submit simulation/PoolBoiling/SingleBubble
```

`Submitting job` with no further output is normal — the run is in progress.
Monitor from a second SSH session:

```bash
ps aux | grep flashx                                                    # is it running?
tail -f ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble/INS_Pool_Boiling.log
ls ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble/*hdf5_plt* 2>/dev/null | wc -l
```

> The `UCX WARN ... UCP version is incompatible` and `ignoring unknown
> parameter` lines are harmless noise on a single-node run.

---

## Phase 16 — Changing simulation parameters

All parameters live in `flash.toml`. After editing, **regenerate inputs**
(Phase 13) — no recompile is needed unless you change `flashOptions.sh`.

Example — shorten the run for a quick test:

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
pkill -f flashx                                       # stop the current run
sed -i 's/^Parfile.tmax = .*/Parfile.tmax = 20/' flash.toml
grep tmax flash.toml                                  # confirm
python3 ../../flashInput.py --input flash.toml        # regenerate flash.par
cp pool_boiling_hdf5_htr_0001 flash_hdf5_htr_0001     # re-copy heater file
cd ~/Outflow-Forcing-BubbleML-main
jobrunner submit simulation/PoolBoiling/SingleBubble
```

Common knobs: `Parfile.ins_gravY` (gravity), `Parfile.ins_invReynolds` (1/Re),
`Parfile.mph_Stefan` (superheat / wall temp), `Heater.numSites` (number of
bubbles), `Parfile.plotFileIntervalTime` (output frequency).

---

## Phase 17 — Convert output to a BubbleML HDF5 dataset

Upload BubbleML (from your **Mac**):

```bash
scp -i BubbleID.pem -r /Users/sarah_yang/Downloads/BubbleML-main ubuntu@132.145.142.183:/home/ubuntu/
```

On **Lambda**, install deps and write a small driver script *inside* the
`scripts/` folder so the import resolves:

```bash
pip3 install boxkit==2023.6.7 torch h5py joblib matplotlib

cat > ~/BubbleML-main/scripts/convert.py << 'PYEOF'
from boxkit_dataset import BoilingDataset
import h5py

sim_dir = "/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble"
output_file = "/home/ubuntu/SingleBubble.hdf5"

print(f"Reading simulation data from: {sim_dir}")
b = BoilingDataset(sim_dir)

perm = (2, 0, 1)
with h5py.File(output_file, 'w') as f:
    f.create_dataset('temperature', data=b._data['temp'].permute(perm))
    f.create_dataset('velx',        data=b._data['velx'].permute(perm))
    f.create_dataset('vely',        data=b._data['vely'].permute(perm))
    f.create_dataset('dfun',        data=b._data['dfun'].permute(perm))
    f.create_dataset('pressure',    data=b._data['pres'].permute(perm))
    f.create_dataset('massflux',    data=b._data['mflx'].permute(perm))
    f.create_dataset('normx',       data=b._data['nrmx'].permute(perm))
    f.create_dataset('normy',       data=b._data['nrmy'].permute(perm))
    f.create_dataset('x',           data=b._data['x'].permute(perm))
    f.create_dataset('y',           data=b._data['y'].permute(perm))

    # Runtime params come from a source plot file, not the output file
    with h5py.File(b._filenames[0], 'r') as src:
        if 'real runtime parameters' in src:
            f.create_dataset('real-runtime-params', data=src['real runtime parameters'][:])
        if 'integer runtime parameters' in src:
            f.create_dataset('int-runtime-params', data=src['integer runtime parameters'][:])

print(f"Done! Written to {output_file}")
PYEOF

cd ~/BubbleML-main/scripts
python3 convert.py
```

> **Bugs fixed:**
> - `ModuleNotFoundError: boxkit_dataset` — running from elsewhere can't find the
>   module. Placing `convert.py` in `scripts/` and running from there resolves it.
> - The stock `to_hdf5` reads runtime params from the *output* file (always
>   empty). This version reads them from the source plot file instead.
>
> The SingleBubble plot-vars include `mflx`, `nrmx`, `nrmy`, so all ten fields
> are written. (The WallSuperheat study omits those — see
> `WALLSUPERHEAT_FULL_SETUP.md`.)

Verify:

```bash
python3 -c "
import h5py
with h5py.File('/home/ubuntu/SingleBubble.hdf5','r') as f:
    print('Datasets:', list(f.keys()))
    print('temperature shape (T,Y,X):', f['temperature'].shape)
"
```

The frame count `T` equals `(number of plot files − 1)` — the converter drops
the final plot file by design.

Download to your Mac:

```bash
scp -i BubbleID.pem ubuntu@132.145.142.183:/home/ubuntu/SingleBubble.hdf5 ~/Downloads/
```

### Output dataset reference

| Dataset | Shape | Meaning |
|---|---|---|
| `temperature` | (T,Y,X) | Non-dimensional temperature [0–1] (×wallTemp for degrees) |
| `velx`, `vely` | (T,Y,X) | Velocity components |
| `dfun` | (T,Y,X) | Signed distance (>0 vapor, ≤0 liquid) |
| `pressure` | (T,Y,X) | Pressure gradient |
| `massflux` | (T,Y,X) | Interfacial mass flux |
| `normx`, `normy` | (T,Y,X) | Interface normals |
| `x`, `y` | (T,Y,X) | Grid coordinates |
| `real-runtime-params` | (N,) | Re, Pr, Stefan, domain size, … |
| `int-runtime-params` | (N,) | nblockx/y/z, tile sizes |

---

## Phase 18 — Visualize

### A. Raw Flash-X output in ParaView (via FlashKit)

On **Lambda**, generate XDMF wrappers:

```bash
cd ~/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble
ls INS_Pool_Boiling_hdf5_plt_cnt_* | wc -l        # how many plot files (sets -e)
flashkit create xdmf -b 0 -e <last_number>
```

Download and open on your **Mac**:

```bash
scp -i BubbleID.pem -r ubuntu@132.145.142.183:/home/ubuntu/Outflow-Forcing-BubbleML-main/simulation/PoolBoiling/SingleBubble ~/Downloads/SingleBubble-output
```

In ParaView (https://www.paraview.org/download/): **File → Open** the `.xmf`
file → **Apply** → choose a variable (`temp`, `dfun`, `velx`, `pres`) → **Play**.
For the exact bubble interface use **Filters → Contour** on `dfun` at level `0`.

### B. Converted HDF5 in Python (matches the BubbleML data_loading.ipynb)

On your **Mac**:

```bash
cd ~/Downloads
cat > viz_bubbleml.py << 'PYEOF'
import h5py, numpy as np, matplotlib.pyplot as plt

with h5py.File('SingleBubble.hdf5','r') as f:
    temp = f['temperature'][:]
    dfun = f['dfun'][:]

t = min(50, temp.shape[0]-1)
fig, ax = plt.subplots(1, 2, figsize=(8,8))
# np.flipud matches the notebook's orientation (heater at the bottom)
im0 = ax[0].imshow(np.flipud(temp[t]), cmap='viridis'); ax[0].set_title(f'Temperature t={t}')
plt.colorbar(im0, ax=ax[0], shrink=0.5)
im1 = ax[1].imshow(np.flipud(dfun[t] >= 0));            ax[1].set_title(f'Bubbles t={t}')
plt.tight_layout(); plt.savefig('bubble_viz.png', dpi=150)
print('Saved bubble_viz.png')
PYEOF
python3 viz_bubbleml.py
```

> **If your plume looks off-center / "cropped":** you ran the **demo** config
> (Phase 12 skipped). The demo uses a slip-wall half-domain, so the bubble sits
> at the left edge. Either redo Phase 12 to get the centered full domain, or for
> a quick visual mirror the half-domain across its left edge:
> `np.concatenate([np.fliplr(field), field], axis=1)`.
>
> **Re-dimensionalize temperature** before comparing across wall temperatures:
> multiply the normalized `temperature` field by the heater temp (e.g. 103).

---

## Appendix — Summary of every change vs. the stock repo

| File / action | Change | Reason |
|---|---|---|
| Flash-X upload | `tar` with `COPYFILE_DISABLE=1` | symlinks + macOS `._*` files |
| `sites/lambda/modules.sh` | one-line stub | no `module` system on Lambda |
| `sites/lambda/Makefile.h.FlashX` | add `-fallow-argument-mismatch` | gfortran 10+ MPI type checks |
| `config.sh` | `SiteName=lambda`, disable AMReX/FlashX/HDF5 | manual installs |
| `software/setup*.sh` | `git@github.com:` → `https://` | no SSH keys |
| AMReX | build tag **23.11** not `cff96a9` | match Flash-X main |
| `flashOptions.sh` | `Ins*` flags, `+InsExtras=True`, single line | renames, `omgm_var`, parse bug |
| `flashBuild.sh` | drop `git checkout afee52b0` | not a git repo |
| `Jobfile` | `sbatch` → `bash` | no SLURM |
| `flash.toml` (Phase 12) | centered domain, noslip wall, FC72 props, Stefan | match published SingleBubble-Saturated dataset |
| heater file | copy to `flash_hdf5_htr_0001` | filename mismatch |
| `flashRun.sh` | `mpirun -np 12` | 64 ranks segfault; 12 divides 108 & 216 |
| `convert.py` | placed in `scripts/`, params from source | import path + empty params |
