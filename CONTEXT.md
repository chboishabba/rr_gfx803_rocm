# Context: rr_gfx803_rocm build flow (torch -> torchvision)

## Goal
Build a ROCm-enabled PyTorch wheel (cached) and then build torchvision against that wheel for gfx803 on Arch-based ROCm 6.4 base images.

## Current State
- Dockerfile: `Dockerfile_rocm64_pytorch_arch_rmake_opt`
- PyTorch build completes and wheel is installed into `/opt/py313`.
- Torchvision build fails at `setup.py bdist_wheel` with:
  `generic_type: cannot initialize type "RpcBackendOptions": an object with that name is already defined`.

## Key Steps (required)
1. Build base image `rocm6_gfx803_base_arch:rmake` first (per README).
2. Create Python 3.13 venv at `/opt/py313`.
3. Clone ROCm PyTorch (`release/2.6`) into `/pytorch`.
4. Apply gloo and CMake version fixes.
5. Build PyTorch using `tools/amd_build/build_amd.py`.
6. Build and install torch wheel from `/pytorch/dist` into `/opt/py313`.
7. Clone torchvision (`release/0.21`) into `/vision`.
8. Build and install torchvision wheel from `/vision/dist`.

## Pitfalls / Gotchas
- **Mixed torch imports**: The torchvision build crash indicates the Python process is loading two different `torch` builds (duplicate type registration). This typically happens if `sys.path` includes a source tree or an extra install path in addition to the installed wheel.
- **PYTHONPATH leakage**: Any `PYTHONPATH` pointing at `/pytorch` (or a previous site-packages) can cause duplicate `torch` loads.
- **Stale build artifacts**: A previous or partial `torch` build in the working directory can shadow the installed wheel.
- **Build caching confusion**: Docker cache can hide which layers actually ran. Make sure the torch wheel install layer is used and no later layer reintroduces another torch copy.

## Recommended Guardrails for Torchvision Build
- Ensure a clean Python path for torchvision build:
  - Use `PYTHONNOUSERSITE=1`.
  - Clear `PYTHONPATH` (empty) for the build command.
- Optional but safe: remove `/pytorch` after installing the torch wheel to prevent accidental imports from the source tree.

## Minimal Fix (torchvision build layer)
Use a clean environment when building torchvision:
- `PYTHONNOUSERSITE=1 PYTHONPATH= /opt/py313/bin/python setup.py bdist_wheel`

Optional: remove `/pytorch` after torch install:
- `RUN rm -rf /pytorch`

## Notes
- Do not skip or replace the PyTorch build; it is already built and should be cached in the Docker layers.
- The failure happens after `Step 44: ** BUILDING TORCHVISION ***`.
