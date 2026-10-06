# FreeRTOS – Middleware (Azure DevOps staging)

Internal Azure DevOps mirror of **FreeRTOS** (upstream: [FreeRTOS/FreeRTOS](https://github.com/FreeRTOS/FreeRTOS), the "classic" distribution), imported here for testing before being exported as a public middleware module to GitHub.

This is the **import** side only — see `FreeRTOS_AzureImport.sh` in `GithubArtifactsHandler`. The export step (Azure DevOps → GitHub, with its own testing/approval logic) is separate and not yet built.

This file itself is regenerated (copied verbatim) by every import run — don't hand-edit it here, edit `README_FreeRTOSDefault.md` next to `FreeRTOS_AzureImport.sh` instead.

---

## Branch & repository layout

Everything lives on a single anchor branch (`master`). One real commit per FreeRTOS version, tree replaced wholesale each time, commit message naming its **normalized** version (e.g. `202212.1.0`, `10.4.1` — see Versioning below). No git tags are created here — this is the Azure DevOps staging side only; tagging happens later, during the separate export step that promotes a tested/approved version to the public GitHub middleware repo.

```
<repo root>
├── CMakeLists.txt              auto-generated every version (see below) — do not hand-edit
├── README.md                   this file
├── FreeRTOSConfig.h.template   copied from this version's own upstream template
├── FreeRTOSConfig.cmake.template  freertos_config target, FREERTOS_PORT, FREERTOS_HEAP
└── FreeRTOS/                   the release, exactly as the FreeRTOSv<version>.zip asset ships it
    └── FreeRTOS/                    upstream's own top-level folder (yes, "FreeRTOS/FreeRTOS" — that's
        │                            how upstream's own zip is laid out, confirmed by extracting a
        │                            real release; the same double segment already appears in older
        │                            hand-written examples like `Middlewares/FreeRTOS/FreeRTOS/Source`)
        ├── Source/                       FreeRTOS-Kernel
        └── ...
    └── FreeRTOS-Plus/
        └── Source/
            ├── FreeRTOS-Plus-TCP/         has a CMakeLists.txt, but the wrapper add_subdirectory()s
            │                              its source/ + tools/ subfolders instead — see below
            ├── AWS/sigv4/                 has its own CMakeLists.txt, add_subdirectory()'d directly
            ├── coreJSON/                  no CMakeLists.txt — only jsonFilePaths.cmake
            ├── Application-Protocols/coreMQTT/   no CMakeLists.txt — only mqttFilePaths.cmake
            └── ...                        whatever this release actually ships — see below
```

## Source

Each upstream FreeRTOS release on GitHub actually offers **three** downloadable files: the release's own pre-packaged `FreeRTOSv<version>.zip` asset, plus GitHub's auto-generated "Source code (zip)" and "Source code (tar.gz)" archives. `FreeRTOS_AzureImport.sh` downloads **only** the pre-packaged `FreeRTOSv<version>.zip` asset (resolved from the release's `assets` list, which excludes the two auto-generated archives).

## Submodules

Upstream `FreeRTOS/FreeRTOS` submodules the kernel (`FreeRTOS/Source`, from `FreeRTOS/FreeRTOS-Kernel`) and several FreeRTOS+ libraries under `FreeRTOS-Plus/Source/` (coreMQTT, corePKCS11, FreeRTOS+TCP, ...). `FreeRTOS_AzureImport.sh` does **not** resolve these via `git clone --recurse-submodules` — the pre-packaged `FreeRTOSv<version>.zip` asset it downloads already ships every one of them fully resolved and flattened, with no `.git` anywhere. What lands in `FreeRTOS/` here is exactly what upstream publishes in that zip — no submodules, no gitlinks.

## CMake support requirement — old releases are not imported at all

Upstream only added CMake build support to the kernel in **FreeRTOS-Kernel V10.5.0** (September 2022, first carried into the classic distribution's own dated releases around `202212.00`). A release older than that has **no `CMakeLists.txt` anywhere** under `FreeRTOS/FreeRTOS/Source`, so there is nothing this repo could publish for it. `FreeRTOS_AzureImport.sh` checks for that file right after extracting the release and, if it's missing, **skips the whole version — no commit, nothing published for it in this repo at all.** This repo's commit history therefore only ever contains versions with a working `freertos_kernel` CMake target.

### Not every `FreeRTOS-Plus/Source/*` folder is a CMake-buildable library — and not every one is even directly under `Source/`

Verified against a real downloaded release (202411.00), `FreeRTOS-Plus/Source/` mixes three different situations, and `GenerateWrapperCMakeLists` handles each on its own terms rather than assuming they're all alike:

1. **Older, vendored, or third-party pieces with no CMake integration at all** — e.g. `FreeRTOS-Plus-CLI`, `FreeRTOS-Plus-IO`, `FreeRTOS-Plus-Trace`, `WolfSSL` (its own autotools build), `mbedTLS` (a full separate upstream project, consumed on its own terms). No marker file anywhere, so these are silently left out entirely — there was never anything to wire up.
2. **Components with their own real `CMakeLists.txt`** — as of 202411.00 this is only `FreeRTOS-Plus-TCP` and `AWS/sigv4` (yes, nested one level inside a plain `AWS/` category folder, alongside `AWS/jobs`, `AWS/ota`, `AWS/device-shadow`, `AWS/device-defender`, `AWS/fleet-provisioning` — those five have no CMakeLists.txt yet, only sigv4 does). These get a real `option(FREERTOS_ENABLE_<NAME>)` + `add_subdirectory()` in the generated file — upstream's own CMakeLists.txt does everything, nothing is invented.
3. **Components that ship only a `<name>FilePaths.cmake`** — this is upstream's own "bring your own build" convention: the file just sets `<PREFIX>_SOURCES` / `<PREFIX>_INCLUDE_PUBLIC_DIRS` CMake variables for you to `add_library()` with yourself; there is no `add_library()` anywhere upstream for these. In 202411.00 this covers **coreJSON, corePKCS11, coreMQTT, coreHTTP, coreSNTP, coreMQTT-Agent, backoff_algorithm**, and the AWS IoT device libraries (`device-shadow`, `device-defender`, `jobs`, `ota`, `fleet-provisioning`) — despite these being the same well-known "AWS core libraries" people usually mean by "FreeRTOS-Plus", none of them ship a ready CMake target in this distribution today. `GenerateWrapperCMakeLists` lists each as a comment (naming the exact `*FilePaths.cmake` to `include()`) instead of guessing an `add_library()` for it — several of these need more than one `*_SOURCES` variable to build correctly (coreMQTT needs `MQTT_SOURCES` **and** `MQTT_SERIALIZER_SOURCES`; corePKCS11 additionally exposes platform-specific `PKCS_PAL_POSIX_SOURCES`/`PKCS_PAL_WINDOWS_SOURCES` that only one of should be compiled), so silently auto-generating this could ship an incomplete or non-portable library without any error. Wire these up yourself, e.g.:

   ```cmake
   include(${CMAKE_CURRENT_SOURCE_DIR}/Middlewares/FreeRTOS/FreeRTOS/FreeRTOS-Plus/Source/Application-Protocols/coreMQTT/mqttFilePaths.cmake)
   add_library(core_mqtt ${MQTT_SOURCES} ${MQTT_SERIALIZER_SOURCES})
   target_include_directories(core_mqtt PUBLIC ${MQTT_INCLUDE_PUBLIC_DIRS})
   ```

The category folders themselves (`AWS/`, `Application-Protocols/`, `Utilities/`) are never components — the search looks one level inside them for the real thing, but no further (so a component's own `test/`, `tools/`, `docs/`, `portable/` subdirectories, which litter this tree with CMakeLists.txt files meant only for upstream's own unit tests, CBMC proofs and Coverity configs, are never picked up as fake components).

---

## Building against it — the generated `CMakeLists.txt`

The root `CMakeLists.txt` in this repo does **not** invent any library — every `add_subdirectory()` in it points at a `CMakeLists.txt` that upstream itself ships. It is auto-generated per version by scanning the tree as described above, so it always matches this release's real layout instead of a hand-maintained list that could drift out of date. The kernel is added unconditionally (everything else needs it or shares its config); every other component is `OFF` by default.

The very first time the generated `CMakeLists.txt` is configured, it also seeds a starter `FreeRTOSConfig.h` **one directory above wherever you vendored this repo** (e.g. next to it, if you put this repo at `Middlewares/FreeRTOS`, the file lands at `Middlewares/FreeRTOSConfig.h`), copied from this version's own `FreeRTOSConfig.h.template` — but only if nothing is already there, so it never overwrites a config you've since edited. Point your own `freertos_config` INTERFACE library at that directory (see step 1 below) and you have something that builds immediately, ready to tune for your target.

Vendor this repo (e.g. as a submodule at `Middlewares/FreeRTOS`), then from **your own** top-level `CMakeLists.txt` — no compiler flags, only plain CMake variables and a target:

```cmake
# 1. Config: point freertos_config at the directory containing your
#    FreeRTOSConfig.h (start from FreeRTOSConfig.h.template in this repo)
#    and, if you enable FreeRTOS+TCP, your FreeRTOSIPConfig.h too.
add_library(freertos_config INTERFACE)
target_include_directories(freertos_config INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/Config)

# 2. Port / component selection — plain cache variables, nothing else needed.
set(FREERTOS_PORT "GCC_ARM_CM4F" CACHE STRING "")            # match your MCU/toolchain
set(FREERTOS_HEAP "4" CACHE STRING "")                        # optional, default heap_4.c
set(FREERTOS_ENABLE_FREERTOS_PLUS_TCP ON CACHE BOOL "")       # only if you need it
set(FREERTOS_PLUS_TCP_NETWORK_IF "STM32" CACHE STRING "")     # only if +TCP is enabled

# 3. Pull in the wrapper, then link whichever real upstream targets you enabled.
add_subdirectory(Middlewares/FreeRTOS)
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        freertos_kernel
        freertos_plus_tcp
)
```

`freertos_config` must exist **before** `add_subdirectory(Middlewares/FreeRTOS)` — the generated `CMakeLists.txt` checks for it up front and fails fast with a clear message if it's missing. `FREERTOS_PORT` must match your MCU/toolchain (see `FreeRTOS/Source/portable/` in this repo for the available port names).

### Known components — verified against a real 202411.00 download

The exact set of `option(FREERTOS_ENABLE_<NAME>)` flags depends on what this particular version's `FreeRTOS-Plus/Source/` actually contains — open the generated `CMakeLists.txt` at repo root to see this version's real list (components without their own CMakeLists.txt appear there as a comment, not an option). As of 202411.00:

| Component | Wiring | Enable flag / target | Depends on |
|---|---|---|---|
| `FreeRTOS/Source` (kernel) | `add_subdirectory()` (always on) | target `freertos_kernel` | `freertos_config` |
| `FreeRTOS-Plus-TCP` | `add_subdirectory()` | `FREERTOS_ENABLE_FREERTOS_PLUS_TCP` → target `freertos_plus_tcp` | `freertos_kernel` + `freertos_config` (must also expose `FreeRTOSIPConfig.h`) + `FREERTOS_PLUS_TCP_NETWORK_IF` |
| `AWS/sigv4` | `add_subdirectory()` | `FREERTOS_ENABLE_SIGV4` → target `sigv4` | none (standalone) |
| `coreJSON` | **manual** — `jsonFilePaths.cmake` only | none generated | none (standalone) |
| `corePKCS11` | **manual** — `pkcsFilePaths.cmake` only | none generated | an mbedTLS target you provide; pick POSIX or Windows PAL sources yourself |
| `Application-Protocols/coreMQTT` | **manual** — `mqttFilePaths.cmake` only | none generated | none (standalone; needs both `MQTT_SOURCES` and `MQTT_SERIALIZER_SOURCES`) |
| `Application-Protocols/coreHTTP` | **manual** — `httpFilePaths.cmake` only | none generated | none (standalone) |
| `Application-Protocols/coreSNTP`, `coreMQTT-Agent` | **manual** — `*FilePaths.cmake` only | none generated | none (standalone) |
| `Utilities/backoff_algorithm` | **manual** — `backoffAlgorithmFilePaths.cmake` only | none generated | none (standalone) |
| `AWS/device-shadow`, `device-defender`, `jobs`, `ota`, `fleet-provisioning` | **manual** — `*FilePaths.cmake` only | none generated | none (standalone) |

**Important:** unlike the standalone libraries above, `FreeRTOS-Plus-TCP` is *not* independent of the kernel — it directly links `freertos_kernel` and shares `freertos_config` with it (which must expose both `FreeRTOSConfig.h` and `FreeRTOSIPConfig.h`). Enabling it without also linking `freertos_kernel` will fail to link.

### `FreeRTOS-Plus-TCP` is wired up specially — its own top-level `CMakeLists.txt` is not meant to be embedded

Extracting a real release and reading `FreeRTOS-Plus-TCP/CMakeLists.txt` shows it does **not** define the `freertos_plus_tcp` library itself. It only calls `project(FreeRTOS-Plus-TCP ...)`, unconditionally `add_subdirectory()`s `source`, `tools` **and** `test`, and even `FetchContent_Declare()`/`FetchContent_MakeAvailable()`s its **own** copies of FreeRTOS-Kernel and CMock over the network. That file is a standalone build-and-test bootstrapper for the FreeRTOS-Plus-TCP repo on its own, not something meant to be `add_subdirectory()`-ed from a larger project — embedding it as-is would silently pull in a second kernel, a test framework, and a live git fetch at configure time.

So the generated wrapper does **not** point at `FreeRTOS-Plus-TCP`'s own root. Instead, verified against a real 202411.00 download and a real end-to-end `cmake --build`, it:

- `add_subdirectory()`s `FreeRTOS-Plus-TCP/source` directly — that's where `add_library(freertos_plus_tcp ...)` actually lives.
- `add_subdirectory()`s `FreeRTOS-Plus-TCP/tools` too — `freertos_plus_tcp_utilities`, a real `PRIVATE` link dependency of `freertos_plus_tcp` itself, lives there and nowhere else. (Confirmed the hard way: the first version of this fix linked fine right up until `-lfreertos_plus_tcp_utilities` couldn't be found.)
- deliberately never adds `test` — that's the one that fetches a second kernel copy and CMock, and builds unit tests that have no business in an embedded consumer build.
- appends `FreeRTOS-Plus-TCP/cmake_modules` to `CMAKE_MODULE_PATH` before adding `source` — the POSIX and WinPCap network-interface backends under `source/portable/NetworkInterface/` call `find_package(PCAP REQUIRED)`, which only resolves via `FreeRTOS-Plus-TCP/cmake_modules/FindPCAP.cmake`. Upstream's own top-level file normally appends this path; since we bypass that file, the wrapper appends it itself.
- requires you to set `FREERTOS_PLUS_TCP_NETWORK_IF` yourself (fails fast with `FATAL_ERROR` if it's unset when `FREERTOS_ENABLE_FREERTOS_PLUS_TCP` is on) — upstream's own top-level file normally defaults/validates this variable with a large `POSIX`/`WIN_PCAP`-detection block; bypassing that file means that defaulting never runs, so the wrapper asks for it explicitly instead of guessing.
- does default `FREERTOS_PLUS_TCP_BUFFER_ALLOCATION` to `"2"` if you don't set it — same reasoning, but this one has a safe upstream default worth keeping.

This is a small, manually curated, explicitly-verified exception — not something this importer auto-detects by scanning CMake source text (reliably telling "this file defines a library" from "this file bootstraps a standalone project" would need an actual CMake parser, not shell text tools). If a future release restructures `FreeRTOS-Plus-TCP`, or another component turns out to need the same treatment, extend `FreeRTOSPlus_GetVerifiedAddSubdirectorySubpaths` in `FreeRTOS_AzureImport.sh` after verifying against a real downloaded release **and** a real end-to-end build, the same way this one was found.

A future release may add a real `CMakeLists.txt` upstream for one of the "manual" rows above (several of these repos already carry one on their own GitHub `main` branch — it just isn't part of what's pinned into this particular vendored release yet); when that happens, `GenerateWrapperCMakeLists` will pick it up automatically as a normal `option()` the next time this importer runs, with no changes needed here.

---

## Notes

- A brand-new repo (or an existing repo that's just missing the `master` branch) gets a real `Initial commit` containing just this `README.md`, pushed immediately when the branch is created — so it's never left with zero commits even if the first version afterwards fails or every available version is soft-skipped. The first version actually imported then wholesale-replaces that with the full tree, same as every later version.
- No git tags are created by this importer — this is the Azure DevOps staging side only. Tagging happens later, during the separate export step that promotes a tested/approved version to the public GitHub middleware repo.
- Re-running the import is idempotent — a version already present as a commit on the anchor branch (matched by its commit message, `FreeRTOS <version> (upstream tag: ...)`) is skipped.
- A release whose `FreeRTOSv<version>.zip` asset was pulled after publishing (e.g. after a security advisory) is soft-skipped with a warning.
- `FreeRTOSConfig.h.template` is copied as-is from that version's own `FreeRTOS/Source/examples/template_configuration/FreeRTOSConfig.h` — it always matches the vendored kernel version. Older releases that don't ship this file will simply have no template published for that version.
- Versions are **normalized** from upstream tags to always be three-part `X.Y.Z`, but with every zero-padded component stripped down to its plain numeric value first — a two-part date-based tag (`YYYYMM.NN`) becomes `YYYYMM.<NN without padding>.0` (e.g. `202212.01` → `202212.1.0`, `202212.00` → `202212.0.0`), and a three-part tag (`X.Y.Z`, with or without a leading `V`) has each component stripped the same way (e.g. `V10.4.1` → `10.4.1`). Unlike `README_FreeRTOSRoot.md` (the GitHub artifact variant), which keeps upstream's own zero-padding (e.g. the old `202212.01.00`), this importer never leaves a padded `01`/`00` component in a version it publishes — a real release counter sitting next to a padded one read as two numbers doing the work of one. A component that already has no leading zero (e.g. a future two-digit counter `11`) is left untouched — stripping never removes a real digit, only padding.

---

## Authors

- **Mr.Nobody** — [embedbits.com](https://embedbits.com)
