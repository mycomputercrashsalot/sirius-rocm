# sirius-rocm

Standalone **ROCm/HIP backend** for [Sirius](https://github.com/sirius-db/sirius) — the GPU-native SQL engine that intercepts DuckDB queries via Substrait and executes them on GPU.

## What this is

Everything needed to build and run Sirius on AMD ROCm/HIP hardware (MI300 / gfx942), without touching the main Sirius codebase. Spun out from [PR #1153](https://github.com/sirius-db/sirius/pull/1153) at Xiangyao Yu's request — maintaining CUDA and ROCm side by side in one repo is a real ongoing cost, so the ROCm backend lives here instead.

### Architecture

```
sirius-rocm/
├── CMakeLists.txt              # Top-level overlay: sets ENABLE_ROCM + delegates to submodule
├── sirius/                     # Git submodule → sirius-db/sirius (pinned commit)
├── patches/
│   └── cuco_compat.hpp                 # cuco API compat header, used by patch_hipdf_cuco.sh
├── scripts/
│   ├── setup.sh                # Init submodule + apply whatever patches/ contains
│   ├── build_rocm_deps.sh      # Build + install hipDF (cuDF for HIP) + hipMM (RMM for HIP)
│   ├── detect_gpu_arch.sh      # Auto-detect AMD GPU arch (asks when none found)
│   ├── patch_hipdf_cuco.sh     # Patch hipDF for hipCollections cuco APIs
│   ├── hipdf_26.06_api_patch.sh # Patch hipDF 25.10 with cuDF 26.06 APIs Sirius needs
│   └── run_tpch_rocm.sh        # End-to-end TPC-H: build → generate data → run 22 queries
├── .github/workflows/
│   └── rocm-test.yml           # CI: shim compile-test on ubuntu-latest
├── PORTING.md                  # Engineering reference: component architecture, shim design, build fixes
└── README.md                   # This file
```

### Dependencies

| Dependency | Source | How consumed |
|---|---|---|
| [sirius-db/sirius](https://github.com/sirius-db/sirius) | Git submodule | Pinned to commit `e88da526` |
| [cuda2rocm](https://github.com/hamstergamerszhang-ops/cuda2rocm) | GitHub | CI clones the default branch; the FetchContent pin lives in the fork's patched CMakeLists |
| [hipDF](https://github.com/ROCm-DS/hipDF) (cuDF for HIP) | Built from source | `build_rocm_deps.sh` → `/opt/rocm` |
| [hipMM](https://github.com/ROCm-DS/hipMM) (RMM for HIP) | Built from source | `build_rocm_deps.sh` → `/opt/rocm` |
| ROCm 7.2.1+ (hipCUB, rocThrust, rocPRIM) | System install | `find_package` / system paths |

## Relationship to sirius-db/sirius and PR #1153

- **PR #1153** is closed; the discussion with the upstream maintainer happened there.
- This repo is the **standalone development home** for the ROCm backend, per the maintainer's request.
- The Sirius source is consumed as a **git submodule** pinned to upstream `sirius-db/sirius`, so there's no code duplication or drift.
- The ROCm source changes (64-bit shuffle masks, CMakeLists ROCm blocks, the cuda2rocm FetchContent pin) live in the [sirius fork's `feat/rocm-port` branch](https://github.com/mycomputercrashsalot/sirius/tree/feat/rocm-port). The `.patch` files this repo's docs originally described were never committed — `setup.sh`'s patch step currently applies nothing, so until the patches are regenerated from the fork, the full-build flow below doesn't complete as written.

## Current status

| Component | Status |
|---|---|
| CUDA→HIP compat shims (cuda2rocm) | ✅ 0 compile errors on gfx942/ROCm 7.2.1 |
| cuco-rocm (bloom_filter, static_set) | 🔧 Compiles in CI; runtime on gfx942 pending — the ~3M keys/s figure was unreproduced (see cuda2rocm's README) |
| hipMM (RMM for HIP) | ✅ Built + installed to `/opt/rocm` on real gfx942 |
| hipDF (cuDF for HIP) | 🔧 Build in progress — 5 hardware-specific fixes applied |
| Sirius end-to-end build | 🔧 Not yet built (blocked on hipDF completion) |
| TPC-H benchmark | ❌ Not yet run |

### Build fixes discovered on real hardware (gfx942 / ROCm 7.2.1)

1. **hipcc as C/CXX compiler** — hipMM's `.cpp` files need `-x hip --offload-arch` (plain `g++` fails)
2. **`git config http.version HTTP/1.1`** — fixes flaky GitHub clones on DSW pods
3. **`ROCM_AMDGPU_TARGETS=gfx942` env** — avoids rapids_cmake architecture mismatch
4. **Pre-clone all 16 CPM deps** — correct branch names from rapids-cmake `versions.json`
5. **Patch `get_rmm.cmake`** — use `find_package(rmm)` instead of CPM re-fetch (avoids rapids_logger conflict)

See `PORTING.md` §8 for full details.

## Quick start

```bash
# Clone
git clone --recursive https://github.com/hamstergamerszhang-ops/sirius-rocm.git
cd sirius-rocm

# Init the Sirius submodule (+ apply whatever is in patches/)
./scripts/setup.sh

# Build hipDF + hipMM (one-time, ~30 min)
./scripts/build_rocm_deps.sh

# Build Sirius with ROCm
cmake -B build -S .
cmake --build build -j$(nproc) --target duckdb sirius_extension

# Run TPC-H
export LD_LIBRARY_PATH=/opt/rocm/lib:${LD_LIBRARY_PATH:-}
./scripts/run_tpch_rocm.sh --scale-factor 1
```

## What's left

Finish the hipDF build on a box that stays up long enough (the 5 fixes above are already applied — the blocker has been rented boxes dying mid-build, not the code). Then build Sirius end-to-end with `ENABLE_ROCM=ON` and run TPC-H for real correctness + performance numbers. Xiangyao's said he's open to hosting this under `sirius-db` once those numbers exist.

## License

Apache-2.0, matching [Sirius](https://github.com/sirius-db/sirius), [cuDF/hipDF](https://github.com/ROCm-DS/hipDF), [RMM/hipMM](https://github.com/ROCm-DS/hipMM), and [cuda2rocm](https://github.com/hamstergamerszhang-ops/cuda2rocm).
