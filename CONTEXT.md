# Context: Full ROCm gfx803 Workflow (Base -> PyTorch -> Torchvision/Torchaudio)

This document summarizes the full build flow used in this repo, with references to the older Ubuntu-based Dockerfiles and the current Arch/rmake flow. It also enumerates known pitfalls.

## Sources (Ubuntu-based references)
- `Dockerfile_rocm64_base` (ROCm 6.4, Ubuntu 24.04, base image)
- `rocm_6.3.4/Dockerfile_rocm634_base` (ROCm 6.3.4, Ubuntu 24.04, base image)
- `rocm_5.4/Dockerfile.base_rocm5.4_source_compile` (ROCm 5.4.2, Ubuntu 22.04, source compile of torch/vision/audio)

## Current Arch/rmake path
- `Dockerfile_rocm64_base_arch_pinned_rmake` (Arch base + ROCm + rmake)
- `Dockerfile_rocm64_pytorch_arch_rmake_opt` (PyTorch -> Torchvision -> Torchaudio)

---

## Full Workflow Overview (Base -> Now)

### 1) Base Image (Ubuntu reference)
From `Dockerfile_rocm64_base` / `rocm_6.3.4/Dockerfile_rocm634_base`:
- Base: `rocm/dev-ubuntu-24.04:*`.
- Set environment for gfx803:
  - `HSA_OVERRIDE_GFX_VERSION=8.0.3`, `PYTORCH_ROCM_ARCH=gfx803`, `ROCM_ARCH=gfx803`.
  - `ROC_ENABLE_PRE_VEGA=1`, `TORCH_BLAS_PREFER_HIPBLASLT=0`, `USE_ROCM=1`, `FORCE_CUDA=1`.
- Install build tools and runtime deps:
  - `git`, `cmake`, `python3`, `python3-venv`, `wget`, `ffmpeg`, `libomp-dev`, `libopenblas-dev`, `ccache`, etc.
- (Optional) install `amdgpu_top` for monitoring.
- (Optional) recompile rocBLAS for gfx803.

### 2) Base Image (Arch/rmake current)
From `Dockerfile_rocm64_base_arch_pinned_rmake`:
- Base: `archlinux:latest` with ROCm pinned paths and rmake flow.
- Similar env exports for gfx803 + ROCm.
- Use `paru` to install build tools and ROCm components:
  - `rocm-cmake`, `rocm-llvm`, `rocm-hip-sdk`, etc.
- Create `.venv` for helper tools (not the main PyTorch venv).

### 3) PyTorch Build (Current)
From `Dockerfile_rocm64_pytorch_arch_rmake_opt`:
- Create Python 3.13 venv at `/opt/py313`.
- Clone ROCm PyTorch `release/2.6` into `/pytorch`.
- Apply CMake and header fixes:
  - `gloo/types.h` add `<cstdint>`.
  - `protobuf`, `psimd`, `FP16`, `NNPACK`, `ittapi`, `gloo`, `ideep` CMake minimum version bumps for CMake >= 4.
- Install build deps (`protobuf` etc.).
- Run `tools/amd_build/build_amd.py`.
- Build wheel with `setup.py bdist_wheel`.
- Install the wheel into `/opt/py313`.

### 4) Torchvision Build (Current)
From `Dockerfile_rocm64_pytorch_arch_rmake_opt`:
- Clone `pytorch/vision` `release/0.21` to `/vision`.
- Build wheel with `setup.py bdist_wheel`.
- Install built torchvision wheel.

### 5) Torchaudio Build (Current)
From `Dockerfile_rocm64_pytorch_arch_rmake_opt`:
- Clone `pytorch/audio` `v2.6.0` to `/audio`.
- Build wheel with `setup.py bdist_wheel`.
- Install built torchaudio wheel.

---

## Known Pitfalls / Failure Modes

### A) Torchvision crash: `generic_type: cannot initialize type "RpcBackendOptions"`
- Cause: The torchvision build process loads two separate `torch` binaries in the same Python process.
- Typical triggers:
  - `PYTHONPATH` or `sys.path` includes `/pytorch` or any other torch source tree.
  - Multiple `torch` installs in different site-packages.
  - Build environment picks up the source tree in addition to the installed wheel.

### B) CMake version incompatibilities
- Newer CMake (>= 4) breaks older subprojects.
- Fixes required in PyTorch tree:
  - `third_party/protobuf/cmake/CMakeLists.txt`
  - `third_party/psimd/CMakeLists.txt`
  - `third_party/FP16/CMakeLists.txt`
  - `third_party/NNPACK/CMakeLists.txt`
  - `third_party/ittapi/CMakeLists.txt`
  - `third_party/gloo/CMakeLists.txt`
  - `third_party/ideep/mkl-dnn/CMakeLists.txt`

### C) ROCm gfx803 quirks
- `HSA_OVERRIDE_GFX_VERSION=8.0.3` and `ROC_ENABLE_PRE_VEGA=1` must be set.
- `TORCH_BLAS_PREFER_HIPBLASLT=0` recommended for this arch.

### D) Docker cache confusion
- PyTorch build is long. The desired flow is to **reuse cached layers** rather than rebuild.
- Ensure the PyTorch build steps remain unchanged to keep cache hits.

---

## Guardrails to Prevent Torchvision Crash

Use a clean environment for the torchvision build:
- Ensure `PYTHONNOUSERSITE=1`.
- Ensure `PYTHONPATH` is empty for the build step.

Optional safety:
- Remove `/pytorch` after installing the torch wheel (prevents accidental source import).

---

## Minimal Fix Summary (Torchvision build layer)
- Build torchvision with clean environment:
  - `PYTHONNOUSERSITE=1 PYTHONPATH= /opt/py313/bin/python setup.py bdist_wheel`

Optional:
- `RUN rm -rf /pytorch` after `pip install dist/torch*.whl`.

---

## Notes
- The torch build **must not be skipped**. It is already built and should be cached.
- The break happens right after `Step 44: ** BUILDING TORCHVISION ***` in the build logs.
