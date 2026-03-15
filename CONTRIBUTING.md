# Contributing

Contributions are welcomed! If you want to contribute anything sizeable, please open an issue first to discuss the change.

External dependencies are vendored via [git-subrepo](https://github.com/ingydotnet/git-subrepo), so there is no need to use submodules, and patching subrepos is easy.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Building](#building)
  - [Linux / macOS (x64)](#linux--macos-x64)
  - [Linux (x32)](#linux-x32)
  - [Cross-compiling for Windows (MinGW)](#cross-compiling-for-windows-mingw)
  - [Building only selected plugins](#building-only-selected-plugins)
  - [CMake options reference](#cmake-options-reference)
- [Testing](#testing)
- [Installing / Deploying](#installing--deploying)
  - [Linux — LADSPA](#linux--ladspa)
  - [Linux — LV2](#linux--lv2)
  - [Linux — VST / VST3](#linux--vst--vst3)
  - [Windows — VST2 with Equalizer APO](#windows--vst2-with-equalizer-apo)
  - [macOS](#macos)
- [Releasing (maintainers)](#releasing-maintainers)

---

## Prerequisites

### All platforms

| Tool | Minimum version | Notes |
|------|----------------|-------|
| [CMake](https://cmake.org/) | 3.6 | Build system |
| [Ninja](https://ninja-build.org/) | any | Recommended generator; `make` also works |
| C++14-capable compiler | — | GCC, Clang, or MSVC |

### Linux (Ubuntu / Debian)

Install the X11 development libraries required by JUCE:

```sh
sudo apt-get install --no-install-recommends \
    libx11-dev libxcomposite-dev libxcursor-dev libxext-dev \
    libxinerama-dev libxrandr-dev libxrender-dev
```

### Cross-compiling for Windows from Linux (MinGW)

```sh
sudo apt-get install mingw-w64
```

> **Note:** MinGW builds currently fail due to certain JUCE incompatibilities. The LADSPA-only build may still work.

---

## Building

### Linux / macOS (x64)

```sh
cmake -Bbuild-x64 -H. -GNinja -DCMAKE_BUILD_TYPE=Release
ninja -C build-x64
```

Built artifacts are placed in `build-x64/bin/`.

### Linux (x32)

```sh
cmake -D CMAKE_CXX_FLAGS=-m32 -D CMAKE_C_FLAGS=-m32 \
      -Bbuild-x32 -H. -GNinja -DCMAKE_BUILD_TYPE=Release
ninja -C build-x32
```

### Cross-compiling for Windows (MinGW)

**x64:**
```sh
cmake -Bbuild-mingw64 -H. -GNinja \
      -DCMAKE_TOOLCHAIN_FILE=toolchains/toolchain-mingw64.cmake \
      -DCMAKE_BUILD_TYPE=Release
ninja -C build-mingw64
```

**x32:**
```sh
cmake -Bbuild-mingw32 -H. -GNinja \
      -DCMAKE_TOOLCHAIN_FILE=toolchains/toolchain-mingw32.cmake \
      -DCMAKE_BUILD_TYPE=Release
ninja -C build-mingw32
```

### Building only selected plugins

By default all plugins supported for a platform are built. You can disable specific plugins:

```sh
cmake -Bbuild-x64 -H. -GNinja -DCMAKE_BUILD_TYPE=Release \
      -DBUILD_VST_PLUGIN=OFF \
      -DBUILD_LV2_PLUGIN=OFF
ninja -C build-x64
```

### CMake options reference

| Option | Default | Description |
|--------|---------|-------------|
| `BUILD_LADSPA_PLUGIN` | `ON` | Build the LADSPA plugin |
| `BUILD_VST_PLUGIN` | `ON` | Build the VST2 plugin |
| `BUILD_VST3_PLUGIN` | `ON` | Build the VST3 plugin |
| `BUILD_LV2_PLUGIN` | `ON` | Build the LV2 plugin |
| `BUILD_AU_PLUGIN` | `ON` | Build the AU plugin (macOS only) |
| `BUILD_AUV3_PLUGIN` | `ON` | Build the AUv3 plugin (macOS only) |
| `BUILD_TESTS` | `ON` | Build and register unit tests |
| `BUILD_FOR_RELEASE` | `OFF` | Enable extra optimisations (LTO) for release builds |
| `BUILD_RTCD` | `OFF` | Enable x86 run-time CPU detection |
| `USE_SYSTEM_JUCE` | `OFF` | Use a system-installed JUCE instead of the vendored copy |
| `BUILD_VERSION` | `1.99` | Version string embedded in plugin metadata |

---

## Testing

Unit tests are built automatically when `BUILD_TESTS=ON` (the default).

After building, run the test suite with [CTest](https://cmake.org/cmake/help/latest/manual/ctest.1.html):

```sh
# Run all tests from the build directory
ctest --test-dir build-x64/src/common/ --output-on-failure
```

Or, run the test binary directly for more detailed output:

```sh
./build-x64/bin/common_plugin_tests
```

The tests use [Catch2](https://github.com/catchorg/Catch2) and exercise:

- `Init → Deinit` lifecycle
- Processing with various channel counts, block sizes, and VAD settings
- Dynamic changes to the retroactive VAD grace period

> The tests are compiled with `-fsanitize=undefined` so undefined behaviour is caught at runtime.

---

## Installing / Deploying

### Linux — LADSPA

Copy (or symlink) `librnnoise_ladspa.so` to a directory on your LADSPA path, typically `/usr/lib/ladspa/` or `~/.ladspa/`:

```sh
sudo cp build-x64/bin/ladspa/librnnoise_ladspa.so /usr/lib/ladspa/
```

Alternatively, use the CMake install target:

```sh
sudo cmake --install build-x64 --prefix /usr
# installs to /usr/lib/ladspa/librnnoise_ladspa.so
```

Verify that the host can find the plugin:

```sh
analyseplugin /usr/lib/ladspa/librnnoise_ladspa.so
```

### Linux — LV2

After building, the `.lv2` bundle is placed under `build-x64/bin/`. Install it with:

```sh
sudo cmake --install build-x64 --prefix /usr
# installs the bundle to /usr/lib/lv2/
```

Or copy manually:

```sh
sudo cp -r build-x64/bin/rnnoise_mono.lv2 /usr/lib/lv2/
sudo cp -r build-x64/bin/rnnoise_stereo.lv2 /usr/lib/lv2/
```

### Linux — VST / VST3

Copy the built plugin to your DAW's VST search path, for example:

```sh
# VST2
mkdir -p ~/.vst
cp build-x64/bin/vst/librnnoise_mono.so ~/.vst/

# VST3
mkdir -p ~/.vst3
cp -r build-x64/bin/rnnoise.vst3 ~/.vst3/
```

### Windows — VST2 with Equalizer APO

1. Download a pre-built release from the [Releases page](https://github.com/werman/noise-suppression-for-voice/releases) **or** build the project on Windows with MSVC and locate `rnnoise_mono.dll` / `rnnoise_stereo.dll` in the `bin\vst\` output directory.
2. Open **Equalizer APO** → **Configuration Editor**.
3. Add a **VST Plugin** effect and point it at the `.dll` file.
4. Make sure your microphone sample rate is set to **48 000 Hz** in *Recording Devices → Properties → Advanced*.

### macOS

macOS builds produce AU and AUv3 plugins in addition to VST3. JUCE copies the built plugins to the system component directories automatically during the build step (`COPY_PLUGIN_AFTER_BUILD TRUE`). If you need to install them manually:

```sh
# AU
cp -r build-x64/bin/rnnoise.component ~/Library/Audio/Plug-Ins/Components/

# VST3
cp -r build-x64/bin/rnnoise.vst3 ~/Library/Audio/Plug-Ins/VST3/
```

---

## Releasing (maintainers)

Releases are automated through GitHub Actions via the **Create Release** workflow (`.github/workflows/release.yml`).

1. Go to **Actions → Create Release** in the GitHub UI.
2. Click **Run workflow** and enter a tag name (e.g. `1.03`).
3. The workflow will:
   - Build the project on **Windows (MSVC)**, **Ubuntu (GCC)**, and **macOS (Clang)** with all plugins enabled and LTO.
   - Run the unit tests on each platform.
   - Package the build artifacts into platform-specific zip files.
   - Create a Git tag `v<tag>` and open a **draft** GitHub Release with all three zips attached.
4. Review the draft release, add release notes, and publish when ready.
