# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PotreeConverter is a C++20 tool that converts LAS/LAZ point cloud files into an octree LOD (Level of Detail) structure for streaming and real-time rendering via [Potree](https://github.com/potree/potree). Version 2.0 outputs only 3 files (metadata.json, hierarchy binary, point data binary) instead of millions, and is 10-50x faster than v1.7.

## Build Commands

```bash
# Build (from repository root)
mkdir build && cd build
cmake ../ && make

# The binary is produced at build/PotreeConverter
# Post-build automatically copies resources/ and license files to the binary directory
```

**Requirements:** CMake 3.16+, C++20 compiler. On Unix/macOS: TBB (Threading Building Blocks) is required.

## Running

```bash
PotreeConverter <input.las> -o <outputDir>
PotreeConverter <input.las> -o <outputDir> -m poisson       # default sampling
PotreeConverter <input.las> -o <outputDir> -m random         # faster, lower quality
PotreeConverter <input.las> -o <outputDir> -m poisson_average
PotreeConverter <input.las> -o <outputDir> --encoding BROTLI # brotli compression
PotreeConverter <input.las> -o <outputDir> -p viewer_name    # generate web viewer page
```

## Architecture

The conversion runs in two sequential phases orchestrated from `main.cpp`:

### Phase 1: Chunking (`chunker_countsort_laszip.cpp/.h`)
Reads LAS/LAZ files and distributes points into temporary octree chunk files using a count-sort algorithm with Morton encoding for spatial partitioning.

### Phase 2: Indexing (`indexer.cpp/.h`)
Reads chunks, applies a sampling strategy to build LOD hierarchy bottom-up, and writes final output files. The sampler is selected via CLI (`-m` flag):
- `SamplerPoisson` (default) — Poisson-disk sampling maintaining minimum point spacing
- `SamplerPoissonAverage` — Poisson with color/attribute averaging
- `SamplerRandom` — random subsampling, fastest

### Key Data Structures (`structures.h`)
- **Node**: Octree node with 8 children, bounding box, point buffer. Named via Morton scheme (e.g., `"r01234567"` where each digit 0-7 selects a child octant). Level = name length - 1.
- **Sampler**: Abstract base class; implementations in `sampler_*.h`
- **Attributes** (`Attributes.h`): Defines point attribute types (position, color, classification, etc.) with scale/offset handling

### Hierarchy Building (`HierarchyBuilder.h`)
Out-of-core construction processes the octree hierarchy in batches grouped by a configurable step size, producing optimized binary hierarchy records.

## Code Layout

```
Converter/
  src/           — main.cpp, indexer.cpp, chunker_countsort_laszip.cpp, logger.cpp
  include/       — All headers (most logic lives in headers due to templates)
  modules/
    LasLoader/   — LAS/LAZ file header parsing
    unsuck/      — Platform abstraction (memory, CPU, file I/O, timing, formatting)
  libs/
    laszip/      — LAZ compression library
    brotli/      — Brotli compression
    json/        — nlohmann/json
    arguments/   — CLI argument parsing
resources/
  page_template/ — HTML/JS template for optional Potree web viewer output
```

## Key Patterns

- Header-heavy design: most logic is in `.h` files (templates, inline implementations)
- Heavy use of `std::execution::par` for parallelism
- Thread safety via `std::mutex`/`lock_guard` and `std::atomic` for progress counters
- `unsuck.hpp` is the utility Swiss-army knife (memory reporting, CPU info, file I/O, formatting)
- Build is always Release mode (`CMAKE_BUILD_TYPE` forced to "Release" in CMakeLists.txt)
- No test framework in C++; test scripts exist in `testing/` (JavaScript/Node.js)
