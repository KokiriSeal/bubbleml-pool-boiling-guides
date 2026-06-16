# Pool Boiling Simulation Guides (BubbleML / Flash-X)

Clear, reproducible **step-by-step instructions** for running custom FC72 pool-boiling
simulations on a fresh Lambda cloud instance using the BubbleML / Flash-X reproducibility
capsule — from a bare machine to a finished BubbleML HDF5 dataset.

> **What this repo is:** documentation only. It does **not** contain or modify the
> simulation code. It is a set of practical, fully worked setup guides I wrote while
> learning to run and extend these simulations, aimed at making them reproducible by
> someone starting from scratch. Every file edit and command is shown explicitly.

## Credit / attribution

All of the underlying simulation software, physics, and design belong to the original
authors. This repository only adds setup documentation on top of their work.

- **Original project:** *Reproducibility Capsule for Multiphase Simulations using Flash-X*
  (the "Outflow-Forcing / BubbleML" repository).
- **Original authors:** Akash Dhruv and Sheikh Md Shakeel Hassan. Copyright © 2023.
- **License:** Apache License 2.0 (see [`LICENSE`](./LICENSE) and [`NOTICE`](./NOTICE)).
- **Related papers:**
  - [A Vortex Damping Outflow Forcing for Multiphase Flows with Sharp Interfacial Jumps](https://arxiv.org/pdf/2306.10174.pdf)
  - [BubbleML: A Multi-Physics Dataset and Benchmarks for Machine Learning](https://arxiv.org/pdf/2307.14623.pdf)
- **Simulation engine:** [Flash-X](https://flash-x.org)

My contribution is limited to the documentation files listed below. Please cite and
credit the original authors and papers for any use of the simulations themselves.

## The guides

| File | What it covers |
|------|----------------|
| [`LAMBDA_SETUP.md`](./LAMBDA_SETUP.md) | Standing up the full toolchain on a fresh Lambda instance. |
| [`WALLSUPERHEAT_SETUP.md`](./WALLSUPERHEAT_SETUP.md) | Quick setup for a wall-superheat pool-boiling run. |
| [`WALLSUPERHEAT_FULL_SETUP.md`](./WALLSUPERHEAT_FULL_SETUP.md) | Full, self-contained wall-superheat walkthrough. |
| [`CUSTOM_STEFAN_NUMSITES_SETUP.md`](./CUSTOM_STEFAN_NUMSITES_SETUP.md) | Choosing a custom Stefan number and nucleation-site count. |
| [`CUSTOM_HOTTER_FULL_SETUP.md`](./CUSTOM_HOTTER_FULL_SETUP.md) | End-to-end "hotter" FC72 run at a custom higher wall temperature. |

## How to use

Start with `LAMBDA_SETUP.md` to build the environment, then follow whichever simulation
guide matches the run you want. Each guide is self-contained and notes where IP addresses
and SSH key filenames must be replaced with your own.

## License

The documentation in this repository is provided under the same Apache License 2.0 as the
original project. See [`LICENSE`](./LICENSE).
