# GMTSAR Repository Architecture Map (Text)

This document is a high-level architecture map of the repository, focused on code organization and execution flow.

## 1) Top-level layout

- `CMakeLists.txt` — root CMake entrypoint; configures dependencies (GMT, TIFF, LAPACK) and wires subprojects.  
- `configure.ac`, `Makefile`, `GNUmakefile`, `config.mk.in` — autotools/make-based build path (legacy + compatibility).  
- `gmtsar/` — core SAR/InSAR processing library and command-line executables.  
- `gmtsar/csh/` — orchestration scripts (`*.csh`) and utility scripts (`.py`) for end-to-end workflows.  
- `preproc/` — satellite/sensor-specific preprocessing pipelines.  
- `snaphu/` — bundled SNAPHU phase unwrapping source and docs.  
- `cmake/` — CMake helper modules and default configuration.  
- `.github/` — CI workflows and issue templates.

## 2) Build/dependency architecture

### Root build graph

`CMakeLists.txt` defines project `GMTSAR` and then adds:

1. `gmtsar` (core library + tools)
2. `snaphu/src` (unwrapping component)
3. `preproc` (preprocessing suite)

Dependencies resolved at root:

- GMT (required)
- TIFF (required)
- LAPACK (required)
- math library (`libm`) when available

### Core module build (`gmtsar/CMakeLists.txt`)

- Builds static/shared library target `gmtsar` from reusable C sources (orbit interpolation, xcorr helpers, IO, filtering, geometry transforms, etc.).
- Builds CLI executables linked against the `gmtsar` library + external libs:
  - `bperp`, `calc_dop_orb`, `conv`, `esarp`, `extend_orbit`, `make_gaussian_filter`,
    `offset_topo`, `phase2topo`, `phasediff`, `phasefilt`, `resamp`,
    `SAT_baseline`, `SAT_llt2rat`, `SAT_look`, `sbas`, `xcorr`.
- Adds `gmtsar/csh` as a subdirectory so workflow scripts are installed/deployed with binaries.

### Preprocessing module build (`preproc/CMakeLists.txt`)

- Currently routes through `ALOS_preproc` in CMake.
- Other preprocessing folders are present and use per-module build scripts/Makefiles, reflecting mixed build systems.

### SNAPHU module

- Source exists under `snaphu/src` with its own `Makefile` and C sources (`snaphu.c`, `snaphu_solver.c`, `snaphu_io.c`, etc.).
- Config templates in `snaphu/config/`; user documentation/manpages in `snaphu/man/`.

## 3) Functional architecture (runtime)

## A. InSAR core computation plane (`gmtsar/`)

Major responsibilities:

- **Geometry/orbit transforms**: llt↔xyz/radar conversions, orbit read/write/interpolate (`SAT_*`, `read_orb.c`, `write_orb.c`, `interpolate_orbit.c`, `geoxyz.c`).
- **Signal processing**: convolution/filtering/correlation (`conv*.c`, `xcorr.c`, `phasefilt.c`, `fft_*`).
- **Phase/topography products**: phase difference, topo correction, offset modeling (`phasediff.c`, `phase2topo.c`, `offset_topo.c`).
- **Time-series/SBAS**: `sbas.c`, parallel helpers, utility functions.
- **Shared infra**: binary formats/structures, parameter parsing, common utilities (`siocomplex*`, `file_stuff.c`, `utils*.c`).

## B. Workflow orchestration plane (`gmtsar/csh/`)

Shell script layer that orchestrates the compiled binaries into full workflows:

- Pair and batch pipelines: `gmtsar.csh`, `batch_processing.csh`, `p2p_*.csh` style scripts.
- TOPS and ScanSAR helpers: alignment/merge/frame creation scripts.
- DEM/topo/geocode helpers and postprocessing utilities.
- External-data integration helpers (e.g., orbit download scripts).

This layer is the operational “glue” between raw data layouts and low-level executables.

## C. Sensor ingestion/preprocessing plane (`preproc/`)

Repository contains dedicated preprocessors for multiple sensors:

- `ALOS_preproc`, `CSK_preproc`, `ENVI_preproc`, `ERS_preproc`, `GF3_preproc`,
  `LT1_preproc`, `NSR_preproc`, `RS2_preproc`, `S1A_preproc`, `TSX_preproc`.

Common pattern per sensor family:

- decode/metadata extraction
- orbit & timing normalization
- swath/frame assembly or stitching
- conversion to SLC products consumed by core `gmtsar` processing

## D. Phase unwrapping plane (`snaphu/`)

SNAPHU is vendored as a separate C codebase and used as the unwrapping backend in interferometric workflows.

## 4) Dataflow map (conceptual)

```text
Raw SAR data + orbit/aux files
        |
        v
preproc/<sensor>/*
  (decode, assemble, normalize -> SLC)
        |
        v
gmtsar core binaries
  (co-registration, interferogram, filtering,
   geometry, topo correction)
        |
        +--> snaphu (phase unwrapping)
        |
        v
gmtsar/csh orchestration scripts
  (batch/pair pipelines, geocoding, export products)
        |
        v
Final interferometric products / time-series outputs
```

## 5) Directory-to-responsibility quick index

- `gmtsar/` → reusable C library + CLI tools (numerical and SAR algorithms)
- `gmtsar/csh/` → end-to-end processing orchestration and operational scripts
- `preproc/` → mission-specific raw-data adapters/preprocessors
- `snaphu/` → phase unwrapping engine
- `cmake/` → build-system modules/defaults
- `.github/workflows/` → CI/testing automation

## 6) Architectural characteristics

- **Layered but polyglot build stack**: CMake + autotools + legacy make coexist.
- **Separation of concerns**:
  - mission-specific ingestion isolated in `preproc/`
  - mission-agnostic SAR math in `gmtsar/`
  - pipeline orchestration in scripts
- **Extensibility model**:
  - new sensors generally plug in under `preproc/<new_sensor>_preproc`
  - new algorithmic kernels fit in `gmtsar/` and are surfaced by scripts
- **Operational orientation**: repo is optimized for executable workflows rather than a single monolithic library API.
