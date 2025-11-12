# SPARTA External Build (vendored source)

This repository contains a **vendored copy of SPARTA** (DSMC simulator) under `sparta/` and instructions
to build the `spa_` binary for both **CPU** and **GPU (Kokkos/CUDA)** targets.

> SPARTA is © Sandia National Laboratories and distributed under the GNU GPL (see `sparta/LICENSE`).  
> We do **not** track compiled artifacts in git; you will generate them locally.

---

## Repository layout

```
opt/
└── sparta/                 # upstream SPARTA source (vendored here for convenience)
    ├── src/                # SPARTA C++ sources and CMake entry
    ├── lib/kokkos/         # Kokkos (includes `bin/nvcc_wrapper`)  <-- present
    ├── examples/           # sample input decks (try these first)
    ├── data/, tools/, doc/ # assets, utilities, documentation
    └── (build-* created locally; ignored by git)
```

Use **out-of-source builds** so the source tree stays clean:

- CPU build dir: `sparta/build-cpu`
- GPU build dir: `sparta/build-gpu`

Both folders are created by you and are **ignored** by git.

---

## Prerequisites

### Common (CPU & GPU)
- Linux (or WSL2). A POSIX shell and build toolchain.
- **CMake ≥ 3.20**
- **C/C++ compiler** (GCC or Clang)
- **MPI** (OpenMPI or MPICH) for parallel runs (`mpirun`, `mpiexec`). Single-rank also works.
- `git`, `make` (or Ninja if you prefer).

Install example (Ubuntu):
```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake openmpi-bin libopenmpi-dev git
```

### Extra for GPU build
- **NVIDIA driver** compatible with your CUDA toolkit
- **CUDA Toolkit** (e.g., 12.x). On many systems this lives at `/usr/local/cuda`.
- A recent GPU (set the correct compute capability / SM arch during CMake).

---

## Quick Start — CPU build (OpenMP backend)

```bash
# from repo root
cd sparta

# fresh CPU build directory
rm -rf build-cpu
cmake -S src -B build-cpu \
  -DKokkos_ENABLE_OPENMP=ON \
  -DKokkos_ENABLE_SERIAL=ON

cmake --build build-cpu -j

# binary will be here:
ls -l build-cpu/src/spa_
```

**Run a sample deck (1 MPI rank):**
```bash
cd sparta
OMP_NUM_THREADS=8 \
mpirun -np 1 ./build-cpu/src/spa_ -in examples/circle/in.circle
```

Notes:
- Increase `OMP_NUM_THREADS` to use more CPU threads on one rank.
- For multi-rank, change `-np` and pick examples that support domain decomposition.

---

## Quick Start — GPU build (CUDA backend via Kokkos)

### 1) Point CMake at CUDA
Most modern CMake/Kokkos flows **do not require** `nvcc_wrapper` explicitly. We’ll let
CMake drive CUDA directly:

```bash
# optional: makes CUDA discovery explicit
export CUDACXX=/usr/local/cuda/bin/nvcc
```

### 2) Configure + build
Replace `89` (Ada SM 8.9) with your GPU’s compute capability.
Common values: `86` (Ampere A10/RTX 30xx), `89` (Ada RTX 40xx), `90` (Hopper H100).

```bash
cd sparta
rm -rf build-gpu
cmake -S src -B build-gpu \
  -DKokkos_ENABLE_CUDA=ON \
  -DKokkos_ENABLE_OPENMP=ON \
  -DKokkos_ARCH_ADA89=ON \
  -DCMAKE_CUDA_ARCHITECTURES=89 \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc

cmake --build build-gpu -j

# binary path:
ls -l build-gpu/src/spa_
```

> **Alternative (nvcc_wrapper):** Some clusters prefer `nvcc_wrapper` as the C++ compiler.  
> Use: `-DCMAKE_CXX_COMPILER=$PWD/lib/kokkos/bin/nvcc_wrapper` and keep `-DKokkos_ENABLE_CUDA=ON`.

### 3) Run on one GPU

SPARTA’s Kokkos build is activated at runtime with `-k on`, `-sf kk`, and `g <num_gpus>`:

```bash
cd sparta
env -u DISPLAY -u XAUTHORITY CUDA_VISIBLE_DEVICES=0 OMP_NUM_THREADS=1 \
mpirun -np 1 ./build-gpu/src/spa_ -in examples/implicit/in.implicit.2d -k on g 1 -sf kk
```

If you have multiple GPUs, set `CUDA_VISIBLE_DEVICES` (e.g., `0,1`) and use `g 2` accordingly.

---

## Choosing the right CUDA architecture

Pick one (and remove the others):

- Ada (RTX 40xx): `-DKokkos_ARCH_ADA89=ON  -DCMAKE_CUDA_ARCHITECTURES=89`
- Ampere (A100/RTX 30xx): `-DKokkos_ARCH_AMPERE80/86=ON  -DCMAKE_CUDA_ARCHITECTURES=80/86`
- Hopper (H100): `-DKokkos_ARCH_HOPPER90=ON -DCMAKE_CUDA_ARCHITECTURES=90`

You can check your GPU’s **SM version** with:
```bash
nvidia-smi --query-gpu=name,compute_cap --format=csv,noheader
```

---

## Useful example decks to try

- `examples/circle/in.circle` — simple 2D case (good first run, CPU or GPU)
- `examples/implicit/in.implicit.2d` — sample implicit/Kokkos deck for GPU

Run syntax recap:
```bash
# CPU (OpenMP)
OMP_NUM_THREADS=8 mpirun -np 1 ./build-cpu/src/spa_ -in examples/circle/in.circle

# GPU (Kokkos/CUDA)
CUDA_VISIBLE_DEVICES=0 mpirun -np 1 ./build-gpu/src/spa_ -in examples/implicit/in.implicit.2d -k on g 1 -sf kk
```

---

## FAQ / Troubleshooting

**Q: GitHub didn’t show my `build-*` folders.**  
A: They’re intentionally ignored. Builds are machine-specific and can exceed GitHub size limits.
Use the build steps above to regenerate binaries locally. If you want the folders to appear *empty*,
drop a sentinel file: `touch sparta/build-gpu/.keep` and `git add -f` it.

**Q: “CMake Warning: No project() command present.”**  
A: You must configure from the **SPARTA CMake entry** (`-S sparta/src`) not the
repo root. Always pass `-S sparta/src -B sparta/build-*` as shown.

**Q: CUDA found, but Kokkos/CUDA kernels don’t run.**  
A: Ensure `-DKokkos_ENABLE_CUDA=ON` and the correct SM architecture flags. At runtime,
use `-k on -sf kk` and set `g <num_gpus>`. Confirm with the banner:
`KOKKOS mode is enabled`.

**Q: “nvcc not found” or architecture mismatch.**  
A: Verify CUDA install (e.g., `/usr/local/cuda/bin/nvcc`) and GPU drivers. Adjust
`CUDACXX` and `-DCMAKE_CUDA_COMPILER` as needed. Match `-DCMAKE_CUDA_ARCHITECTURES`
to your GPU’s SM version.

**Q: MPI “Authorization required” spam.**  
A: Some remote desktops forward X11. We explicitly clear GUI env vars in examples:
`env -u DISPLAY -u XAUTHORITY ...` to silence X11 attempts during `mpirun`.

**Q: How do I speed up CPU runs?**  
A: Increase `OMP_NUM_THREADS` (per rank) and/or increase `-np` for more MPI ranks.
Performance is deck-dependent; domain decomposition quality matters.

**Q: Can I install (make install) instead of running from build tree?**  
A: Yes. Use `-DCMAKE_INSTALL_PREFIX=<dir>` and run `cmake --build <build> --target install`.
Then call `<prefix>/bin/spa_` in your runs.

---

## Repro scripts (optional)

Create convenience scripts if you like:

**`scripts/build_cpu.sh`**
```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT/sparta"
rm -rf build-cpu
cmake -S src -B build-cpu -DKokkos_ENABLE_OPENMP=ON -DKokkos_ENABLE_SERIAL=ON
cmake --build build-cpu -j
```

**`scripts/build_gpu.sh`**
```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT/sparta"
rm -rf build-gpu
cmake -S src -B build-gpu   -DKokkos_ENABLE_CUDA=ON   -DKokkos_ENABLE_OPENMP=ON   -DKokkos_ARCH_ADA89=ON   -DCMAKE_CUDA_ARCHITECTURES=89   -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc
cmake --build build-gpu -j
```

---

## License

SPARTA is distributed under the **GNU General Public License**. See `sparta/LICENSE`.
This README and repository scaffolding are provided under the MIT License unless noted otherwise.
