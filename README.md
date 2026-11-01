# obs-browser-nvidia

Experimental NVIDIA GPU acceleration for OBS browser sources on Linux (Wayland), built on
[obsproject/obs-browser](https://github.com/obsproject/obs-browser) 2.27.5.

Not an official OBS build. Upstream documentation: [docs/UPSTREAM-README.md](docs/UPSTREAM-README.md).

**Current status: experimental test candidate.** CEF 156 has a known missing NVIDIA producer
synchronization dependency ([CEF #4313](https://github.com/chromiumembedded/cef/issues/4313));
consumer `glFinish` does not fix it. Tearing or incomplete frames remain possible. The NVIDIA and
streaming measurements below are inherited historical records, not verification of the current
revision. Current local checks use AMD hardware and validate compilation, CPU rendering and
lifecycle behavior. See [validation](docs/VALIDATION.md) and the
[upstream CEF synchronization patch/source-build guide](docs/CEF-NVIDIA-SYNC.md).

**CPU compatibility correction (2026-11-01):** the first delivered archive
`obs-browser-nvidia-arch-x86_64-eba2d30-dirty-de43e3a638a5.tar.zst` has an
`obs-browser-page` helper declaring **x86-64-v4** and is unsuitable for the Xeon E5-2680 v4
(x86-64-v3). The host glibc `Scrt1.o` supplied that requirement; passing `-march` alone does
not replace incompatible startup objects. Preserve that archive only as an invalid-build record.
The user identifies the working NVIDIA/DMA-BUF measurements as coming from installed **r755 (v3)**,
not from this archive. Its reported log is absent from this checkout, so that attribution is
user-provided evidence rather than a locally repeated target test.

New packages enforce a maximum declared ISA of x86-64-v3 for every bundled ELF file and
record their notes/hashes in `BUILD-INFO.json`. Missing notes are reported as unknown: this
check does not prove opcode compatibility or check the recipient's system libraries. The corrected
build uses generic Arch startup objects and explicit v3 flags. Its helper entry point passed a
QEMU Haswell check, and full OBS passed native Wayland CPU rendering on the AMD host; this is
not a target NVIDIA test. See [current v3 verification](docs/VALIDATION.md#corrected-v3-build-verification--2026-11-01).

## Status

| Feature | State |
|---|---|
| NVIDIA DMA-BUF browser sources, native Wayland | Historical successful runs; producer synchronization unresolved |
| Automatic CPU fallback per source | Implemented; current lifecycle regression checks use software rendering |
| Live streaming with overlays (local RTMP) | Historical measurements only; not rerun for current revision |
| Native Wayland browser docks | Not supported (needs unmerged CEF #4233) |
| Premultiplied sRGB over white | Inherited upstream defect, not fixed |
| Hardware other than RTX 5060 / driver 615 | Not tested |

## Historical NVIDIA test system

| | |
|---|---|
| OS | CachyOS (Arch-based), kernel 7.3.0-4-cachyos |
| Session | Hyprland, native Wayland |
| CPU | Intel Xeon E5-2697A v4, 32 threads |
| RAM | 31 GiB |
| GPU | NVIDIA GeForce RTX 5060, driver 615.78.08 |
| OBS | 33.0.0 |
| CEF | 156.0.14 (plugin, private runtime); 154 (stock package) |

## Historical streaming test

Scene: `01 · Начало` from a real collection. It contains **10 browser overlays**: 9 visible,
1 hidden. Capture sources (screen, windows, microphone, PipeWire audio) were disabled in the
test copy. Stream: 1920×1080 60 FPS, x264 `veryfast`, 6000 kbps, AAC.
Output went to a **local RTMP receiver**, never to a live service.

Single stream, 60 seconds:

| Browser path | Accepted | Output dropped | Render dropped | Avg render |
|---|---|---|---|---|
| Fork, NVIDIA GPU | yes | 0 | 90 / 4475 | 2.68 ms |
| Fork, CPU paint | yes | 0 | 89 / 4472 | 3.51 ms |
| Stock obs-browser, NVIDIA blocked | yes | 0 | 85 / 4473 | 2.99 ms |

Five parallel streams per preset, 60 seconds each (tree CPU = OBS, CEF renderers and launcher):

| Preset | Tree CPU | Output dropped per stream | Render dropped per stream | DMA-BUF |
|---|---|---|---|---|
| Fork, NVIDIA GPU | ~314% | 0–45 (avg 11) | ~177 / ~4600 | yes |
| Fork, CPU paint | ~560% | 1108–1510 (avg 1311) | ~407 / ~4700 | no |
| Stock, NVIDIA blocked | ~568% | 435–1700 (avg 996) | ~369 / ~4700 | no |

In this historical run, the GPU path used about 45% less CPU and dropped far fewer output frames
under five parallel 1080p60 streams. CPU paint and stock behaved about the same. These measurements
are not a current performance guarantee or evidence that producer synchronization is correct.
Single runs are noisy; treat the single-stream render times as indicative only.
Five parallel 1080p60 CPU streams already saturate this machine; do not expect live use at that load.

## Requirements

- Arch Linux or derivative, x86-64-v3 CPU or newer, OBS Studio 33, Qt 6, Wayland session
- NVIDIA driver with GBM/EGL DMA-BUF support
- `bubblewrap` (used for a temporary mount overlay, not as a security sandbox)

## Install

### AUR

```sh
yay -S obs-browser-nvidia-git
```

The AUR package is not published yet. Once it is, this is the supported path.

### Build from a local clone

```sh
git clone -b experimental/nvidia-cef156 https://github.com/Satont/obs-browser-nvidia.git
cd obs-browser-nvidia/packaging/aur
makepkg -si
```

The repository is currently private; cloning requires an authenticated GitHub account with access.
To use an existing local checkout as the package source:

```sh
OBS_BROWSER_FORK_URL=file:///absolute/path/to/obs-browser makepkg -si
```

### Build and stage manually

For local development. When preparing a build for another CPU, use the clean container workflow
below: the host's optimized startup objects can override the compatibility of the result.

```sh
sudo pacman -S --needed base-devel git cmake ninja obs-studio qt6-base nlohmann-json simde \
  libdrm libglvnd libx11 wayland curl bubblewrap ffmpeg
cef=$(bash scripts/fetch-cef.sh)
cmake -S . -B build -G Ninja -DCEF_ROOT_DIR="$cef" \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr
cmake --build build -j "$(nproc)"
DESTDIR="$PWD/stage" cmake --install build
```

`DESTDIR` stages the files under `stage/usr/lib/obs-browser-nvidia`. Nothing in the system
directories or the OBS package files is overwritten. The staged launcher is
`stage/usr/bin/obs-nvidia-browser`; installing with CMake also installs its desktop entry.

### Build a test bundle to send to a friend

Use Docker to build against generic Arch startup objects, independent of an optimized host glibc:

```sh
bash scripts/build-arch-v3-bundle.sh
```

This downloads the pinned **prebuilt** CEF runtime, builds with
`-march=x86-64-v3 -mtune=generic`, runs the startup/fallback checks and a QEMU Haswell helper-entry
check, then packages the Release binaries with test hooks and DevTools disabled. It does not
rebuild CEF/Chromium. Build and check outputs are in `build-v3/`; archives are in `build-v3/dist/`.
Docker must be available to the current user. To reuse an extracted CEF, add `--cef-root /absolute/path`.

Packaging rejects bundled ELF files declaring requirements above x86-64-v3 before writing an archive.
It records compiler flags and per-file ISA metadata/hashes. For an already built compatible runtime,
`bash scripts/package-test-build.sh /absolute/path/to/build /absolute/path/to/dist` performs only
packaging; passing compiler flags alone cannot make a v4 glibc startup object compatible.

Send the `.tar.zst` and `.sha256` from `build-v3/dist/`. The archive contains the plugin, helper,
complete private CEF runtime, data, licenses, corresponding plugin source, and build provenance.
It requires **Arch/CachyOS x86_64 with OBS 33, Qt 6, `qt6-wayland` and `bubblewrap` installed**;
OBS and system libraries are not bundled. It does not target Flatpak OBS or arbitrary Linux ABIs.

The recipient can extract it and run `./obs-nvidia-browser` inside the extracted folder.
This launcher keeps OBS settings and CEF cache in the bundle's `config/` directory. Create a
test scene or import a copy of the scene collection, enable **Browser Hardware Acceleration**,
and restart the test OBS. `OBS_BROWSER_TEST_CONFIG_HOME=/absolute/path` selects another isolated
config. The installed system launcher continues to use normal OBS settings.

Compare the same scene in native Wayland, saving logs and the OBS process exit status:

```sh
OBS_BROWSER_GPU=nvidia OBS_BROWSER_DIAGNOSTICS=1 ./obs-nvidia-browser >gpu.log 2>&1
OBS_BROWSER_GPU=cpu OBS_BROWSER_DIAGNOSTICS=1 ./obs-nvidia-browser >cpu.log 2>&1
```

Look for `accelerated DMA-BUF frame` per visible source. `CPU paint fallback` means it reloaded
on CPU; `NVIDIA shared textures: experimental` alone does not prove GPU rendering. Compare the
CPU use of OBS **and its CEF child processes**, overlay motion/transparency/audio, scene changes,
resize and shutdown. The bundle includes a short Russian/English `README.txt` for the tester.

CEF 156 has an unresolved [NVIDIA producer-synchronization issue](https://github.com/chromiumembedded/cef/issues/4313)
(#4313/#4314): consumer `glFinish` does not establish the missing producer dependency. This test
build may tear or display incomplete frames; prior successful runs do not establish stable or
tearing-free operation on another GPU/driver.

## Run

```sh
obs-nvidia-browser
```

Or pick **OBS Studio (NVIDIA)** from the menu. Native Wayland uses NVIDIA by default;
on X11 it uses CPU paint.

| Variable | Values | Meaning |
|---|---|---|
| `OBS_BROWSER_GPU` | `nvidia`, `cpu`, `software`, unset | NVIDIA shared textures / CPU paint / no Chromium GPU / upstream auto |
| `QT_QPA_PLATFORM` | `wayland`, `xcb` | Qt platform; explicit values are kept |
| `OBS_BROWSER_PROXY` | URL | Chromium proxy for browser sources only |
| `OBS_BROWSER_DIAGNOSTICS` | `1` | Extra logs |

The GPU path needs **Browser Hardware Acceleration** enabled in OBS settings.
The launcher sets `QT_NO_GLIB=1` to avoid a GTK/Wayland startup race seen on this machine.

## Fallback

Each source switches once to CPU paint after an import failure, invalid DMA-BUF data,
renderer termination, or no first frame within 15 seconds. The page reloads and audio may pause
briefly. Other sources and the stream keep running.

## Known limits

- Native Wayland docks are unavailable; use `QT_QPA_PLATFORM=xcb OBS_BROWSER_GPU=cpu` for docks.
- Semi-transparent sRGB over white is still incorrect (inherited from upstream). Example: 50% green
  over white renders as `(187,220,187)` instead of `(128,255,128)`.
- Chromium runs with `no_sandbox`, as upstream does.
- A GPU hang after a successful first frame may not recover automatically.
- Only RTX 5060 / driver 615.78 was tested.

## Tests

```sh
cmake -S . -B build -DOBS_BROWSER_BUILD_TESTS=ON && cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
DESTDIR="$PWD/stage" cmake --install build
python3 tests/benchmark.py                 # GPU/CPU pixels, FPS, CPU tree (native Wayland)
python3 tests/stream_audit.py --gpu-mode nvidia --name my-gpu   # local RTMP stream
python3 tests/stream_audit.py --stock --name my-stock           # stock obs-browser with NVIDIA blocked
```

Results go to `test-results/`, which is git-ignored. Stream tests use isolated OBS profiles and a local
RTMP receiver. They never use your stream key or start a live output.

More detail: [docs/VALIDATION.md](docs/VALIDATION.md) · [docs/FORKS.md](docs/FORKS.md) ·
[docs/IMPLEMENTATION-MATRIX.md](docs/IMPLEMENTATION-MATRIX.md)

## License

GPL-2.0-or-later (upstream obs-browser). CEF is BSD-3-Clause; see `COPYING` and the CEF license file
in the package. Upstream copyright notices are unchanged.
