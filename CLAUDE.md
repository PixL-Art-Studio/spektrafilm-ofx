# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Native OpenFX plugin (DaVinci Resolve, Nuke…) porting the Python `spektrafilm` film simulation to GPU:
Metal on macOS, Vulkan on Windows and Linux x86_64. C++17 / Objective-C++, CMake ≥ 3.24.

The repo is designed to live **inside the upstream `spektrafilm` Python repo at `OFX/SpektraFilm`**. Several paths
resolve relative to that parent (`../../.venv`, `../../src/spektrafilm`, `../../docs/user_manual/main.pdf`,
`../../website/public/downloads`). Standalone checkouts still build (falls back to `find_package(Python3)` and
`documentation/spektrafilm_reference_guide.pdf`), but the Python-reference parity tools need the parent repo.

## Build

Prerequisites: Python with `numpy scipy colour-science` (checked at configure time — build-time table generation),
OpenFX SDK 1.5.1 headers at `third_party/openfx/` (gitignored) or via `OFX_HEADER_DIR` / `-DSPEKTRAFILM_OFX_HEADERS=`.
Windows/Linux need the Vulkan SDK (`VULKAN_SDK`, `glslc` or `glslangValidator`); macOS needs Xcode Metal toolchain + libpng.

```powershell
# Windows (outputs build-windows/, Contents\Win64 bundles)
.\build_windows.ps1                       # -Generator Ninja, -Configuration Debug, -Install
.\build_windows.ps1 -RunStandardHarness   # build + Vulkan smoke harness (see below)
.\tools\package_windows_release.ps1       # public release ZIP (install.bat + bundles + legal)
```

```sh
./build_macos.sh                          # outputs build-release/, release flags preset
./tools/package_macos_release.sh --build-dir build-release   # signed+notarized pkg (needs SPEKTRAFILM_DEVELOPER_ID_* env)
# Linux: plain cmake -S . -B build && cmake --build build --parallel
```

Notable CMake options: `SPEKTRAFILM_OFX_METAL_GPU_BUFFERS` (ON), `SPEKTRAFILM_RELEASE_LTO` (ON),
`SPEKTRAFILM_NATIVE_FAST_MATH` / `SPEKTRAFILM_METAL_FAST_MATH` (OFF — only enable after parity validation),
`SPEKTRAFILM_METAL_PROFILE_SOURCES` (never for release). Versions are per-platform cache vars at the top of
`CMakeLists.txt` (`SPEKTRAFILM_MACOS_VERSION`, `SPEKTRAFILM_WINDOWS_VERSION`).

## Tests / validation

- Python tests (pytest/unittest; they mostly regex-check shader/C++ sources for consistency, no GPU needed):
  `python -m pytest tests tools/test_*.py` — single test: `python -m pytest tests/test_sdr_transfer_encode.py -k <name>`.
- Vulkan smoke harness (`SpektraVulkanCopyHarness`, built on Windows/Linux): flags compose per pipeline stage,
  e.g. `--core-pass --print-scan-pass --preview-grain-pass`, `--halation-pass`, `--diffusion-pass`… The
  `build_windows.ps1 -Run*Harness` switches wrap these.
- macOS only: `SpektraFilmParityHarness`, `SpektraFilmPerfHarness`, `SpektraFilmEvaluationHarness`,
  `SpektraVariantGenerator`. `tools/run_parity_harness.py --output <dir> [--mode quick|full] [--stages ...] [--patterns ...]`
  compares Metal output to the Python reference per stage (`film_log_raw`, `film_density_cmy`, `print_log_raw`,
  `print_density_cmy`, `final_linear_rgb`…). `tools/run_perf_quality_sweep.py` + `tools/perf_candidates_*.json`
  sweep optimization flags against quality. Sample reports live in `PnP/Parity` and `PnP/Performance`.
- Any numeric/shader change must keep Metal and Vulkan behaviour in sync; validate with parity before release.

## Architecture

- **Three plugins from one source**: `CMakeLists.txt` calls `add_spektra_ofx_plugin` three times, compiling
  `src/SpektraFilmPlugin.cpp` with `SPEKTRAFILM_PLUGIN_FLAVOR` = 0 `spektrafilm flow` (simplified UI),
  1 `spektrafilm` (pro), 2 `spektrafilm dev` (not packaged for release). Flavor controls parameter visibility tiers,
  not the render pipeline.
- **`SpektraFilmPlugin.cpp`** (OFX C API, no Support library wrappers): parameter definitions, UI/visibility logic,
  OFX host interaction, `.spkpreset` presets (stored in `~/Documents/spektrafilm/presets`), and translation of
  OFX params into `RenderParams` (`SpektraParameters.h`). Tooltips are in `SpektraTooltips.h`.
- **Renderer abstraction** (`SpektraRenderer.h`): `Renderer::render(source, destination, window, params, time)` plus
  rich `RendererDiagnostics`. `createNativeRenderer()` is defined exactly once per platform —
  `SpektraMetalRenderer.mm` (macOS) or `SpektraVulkanRenderer.cpp` (Windows/Linux); only one is compiled.
- **Shaders**: Metal is one large `shaders/SpektraFilm.metal` → `SpektraFilm.metallib`; Vulkan is split per stage in
  `shaders/vulkan/*.comp` → SPIR-V (`SpektraFilmVulkanShaders` target). Pipeline stages (both backends):
  film exposure → curve develop/DIR couplers → grain → halation → diffusion → print/scan → scanner post.
- **Build-time generated data**: `tools/generate_profile_curves.py` reads `Resources/data/{profiles,filters,luts}`
  (+ local-only, gitignored SMPTE `standards/`) and emits `build/generated/SpektraGeneratedProfileCurves.cpp`,
  `SpektraGeneratedProfileCounts.h` and `.f32` LUTs that are compiled into the plugin. A second copy
  (`generated_parity/`) is built from `profiles/archive` for parity harnesses. Stock lists live in
  `tools/ofx_stock_lists.py`; adding a film/paper means touching profiles JSON + that list, not C++ tables.
- **`SpektraAppBridge.{h,mm}`** (macOS): C API (`SpektraRendererRef`, flavor/param kinds) wrapping the renderer for
  non-OFX consumers; linked into `SpektraFilmNativeCore` with the Metal harnesses.
- `archive/` holds retired code (old profile store/data compiler) — not built.
