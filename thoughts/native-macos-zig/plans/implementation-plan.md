# Native macOS Support for the Zig Build Implementation Plan

## Overview

Add explicit native macOS support to the `zig16` branch's minimal static GLib build so it can produce `libglib-2.0.a` and `libintl.a`, link through the pinned `cataggar/libslirp` consumer, and preserve the existing Linux glibc, Linux musl, and Windows/mingw behavior.

The implementation will follow the platform decisions made by upstream Meson and Homebrew, but will deliberately remain a core-only build. Cocoa, Carbon, `gosxutils.m`, and their Apple framework dependencies are tracked separately in [issue #2](https://github.com/cataggar/glib/issues/2).

## Progress

- [x] Phase 1: Make the pinned gettext runtime Darwin-safe
- [x] Phase 2: Add the minimal Darwin build path to GLib
- [ ] Phase 3: Add static smoke tests and native macOS CI
- [ ] Phase 4: Regression validation and rollout

## Current State Analysis

- `build.zig` treats every non-Windows target as Linux-like. Target classification only distinguishes Windows, musl, and glibc, then warns and continues for other operating systems (`build.zig:165-184`).
- `glib_unix_sources` includes Linux-only `gjournal-private.c`, while upstream Meson adds it only when `host_system == 'linux'` (`build.zig:109-117`, `glib/meson.build:383-394`).
- The generated `glibconfig.h` uses Linux ABI values for all non-Windows targets:
  - `gint64` is `long`, but Apple defines `int64_t` as `long long` (`build.zig:267-276`, `meson.build:1643-1776`).
  - `GLIB_SYSDEF_AF_INET6` is 10 instead of Darwin's 30 (`build.zig:430-437`, `meson.build:2047-2055`).
- The generated `config.h` enables Linux-only headers and APIs under `#ifndef _WIN32`, including `sys/prctl.h`, `sys/auxv.h`, `ppoll`, `getauxval`, `getresuid`, `eventfd`, `SOCK_CLOEXEC`, `pthread_condattr_setclock`, the two-argument `pthread_setname_np`, `clock_nanosleep`, and `pthread_getaffinity_np` (`build.zig:516-588`).
- Darwin APIs required by the core sources are not enabled, including `crt_externs.h`, `_NSGetEnviron`, `mach/mach_time.h`, `pthread_cond_timedwait_relative_np`, and the one-argument `pthread_setname_np`.
- `needs_libintl` excludes macOS, so `libintl.h` is unavailable and the vendored static intl artifact is not built (`build.zig:167-176`, `build.zig:696-764`).
- The pinned gettext runtime has two additional Darwin problems:
  - its hand-written `config.h` defines `HAVE_MEMPCPY`, but macOS has no `mempcpy`;
  - `libgnuintl.h` redirects `setlocale()` to `libintl_setlocale()` on macOS, but `setlocale-compat.c` is only compiled for Windows.
- A native arm64 macOS `zig build -Dtarget=native -Doptimize=ReleaseFast` currently fails with 29 errors before producing either static archive.
- The pinned `cataggar/libslirp` `zig16` build links `glib_dep.artifact("glib-2.0")` and already has a `slirp_new` smoke test. It is therefore the correct end-to-end validation of Zig transitive static linkage.

## Desired End State

- Native arm64 macOS builds and installs:
  - `lib/libglib-2.0.a`
  - `lib/libintl.a`
  - the existing GLib and libintl headers
- The GLib artifact links its vendored intl artifact and the macOS SDK's iconv library through the Zig dependency graph.
- Darwin selects only portable Unix sources and accurate Darwin feature macros.
- `glibconfig.h` matches the target ABI for Apple integer types and socket constants.
- `zig build test` runs a native static GLib smoke test without Homebrew GLib or gettext.
- CI builds the existing Linux glibc, Linux musl, and Windows/mingw targets and runs native macOS GLib and libslirp smoke tests.
- The `cataggar/libslirp` pin is updated after the GLib commit is available, allowing QEMU's later selective macOS libslirp work to consume it.

### Key Discoveries

- Homebrew's GLib formula uses upstream Meson probes rather than Darwin source patches, installs both shared and static libraries, depends on gettext at runtime on macOS, and patches `glib-2.0.pc` so consumers can find `libintl`.
- Homebrew's static metadata includes `-lintl -liconv` and the frameworks enabled by its full Cocoa/Carbon build. This plan adopts the intl/iconv pattern but excludes the optional framework path.
- Upstream GLib uses `pthread_cond_timedwait_relative_np` and one-argument `pthread_setname_np` on Darwin (`meson.build:2145-2202`, `glib/gthread-posix.c:380-510`, `glib/gthread-posix.c:780-832`).
- The core sources use `_NSGetEnviron`, not `_NSGetArgc`; Darwin configuration must define `HAVE_CRT_EXTERNS_H` and `HAVE__NSGETENVIRON` (`glib/genviron.c:30-35`, `glib/genviron.c:325-333`, `glib/gspawn-posix.c:41-42`, `glib/gspawn-posix.c:101-102`).
- macOS provides `O_CLOEXEC` but not `SOCK_CLOEXEC`, `ppoll`, `getresuid`, `eventfd`, `clock_nanosleep`, `pthread_condattr_setclock`, or `pthread_getaffinity_np`.
- A raw static archive cannot encode linker flags. Within Zig, `linkLibrary()` and `linkSystemLibrary()` must propagate them; later non-Zig sysroot consumers such as QEMU must continue to publish matching pkg-config metadata.

## What We Are NOT Doing

- Enabling Cocoa, Carbon, `gosxutils.m`, Foundation, CoreFoundation, AppKit, Carbon, or CoreServices. That optional work is issue #2.
- Building GObject, GIO, GModule implementations, PCRE2-backed regex support, or the full upstream GLib surface.
- Cross-compiling macOS artifacts from Linux.
- Fully statically linking Apple system libraries; `/usr/lib/libiconv` and `libSystem` remain system dependencies.
- Modifying QEMU's release workflow or completing `cataggar/qemu#34` in this change.
- Replacing the hand-maintained Zig configuration with a Meson invocation.

## Implementation Approach

Use explicit target classification and platform-specific configuration blocks instead of expanding the existing "not Windows" branches. Fix the pinned gettext package first, then update GLib's dependency pin and Darwin path. Validate the final link through both a GLib smoke executable and the pinned libslirp consumer.

## Phase 1: Make the Pinned Gettext Runtime Darwin-Safe

### Overview

Fix the minimal static intl package at its source so GLib can reuse the corrected pinned dependency without carrying a private gettext patch.

### Changes Required

#### 1. Correct the Darwin `mempcpy` configuration

**Repository**: `cataggar/gettext`  
**File**: `gettext-runtime/intl/config.h`

- Define `HAVE_MEMPCPY` only where the function exists.
- Leave the macro undefined on Apple rather than defining it as 0. The gettext sources mix `#ifndef HAVE_MEMPCPY` and `#if !HAVE_MEMPCPY`, so defining it as 0 suppresses the forward declaration while enabling the fallback implementation.

```c
#ifndef __APPLE__
#define HAVE_MEMPCPY 1
#endif
```

#### 2. Compile the existing setlocale shim on macOS

**Repository**: `cataggar/gettext`  
**File**: `build.zig`

- Add `is_macos`.
- Compile `setlocale-compat.c` for Windows or macOS.
- Update the comments to describe both platforms.

```zig
const is_macos = target.result.os.tag == .macos;

if (target.result.os.tag == .windows or is_macos) {
    // add setlocale-compat.c
}
```

#### 3. Update GLib's pinned gettext revision

**Repository**: `cataggar/glib`  
**File**: `build.zig.zon`

- After the gettext fix is committed, update the dependency URL and hash with Zig's package tooling:

```sh
zig fetch --save=gettext \
  https://github.com/cataggar/gettext/archive/<commit>.tar.gz
```

### Success Criteria

- `cataggar/gettext` builds `libintl.a` natively on arm64 macOS with Zig 0.16.
- `nm` shows `libintl_setlocale` in the archive.
- No Homebrew headers or libraries are required.
- GLib's `build.zig.zon` points to the corrected immutable gettext commit.

**Implementation Note**: Land or otherwise make the gettext commit fetchable before updating the GLib pin.

---

## Phase 2: Add the Minimal Darwin Build Path to GLib

### Overview

Make source selection, generated headers, and static dependencies explicitly aware of `.macos`.

### Changes Required

#### 1. Replace implicit Unix/Linux target classification

**File**: `build.zig`

- Introduce:
  - `is_linux`
  - `is_macos`
  - `is_windows`
  - `is_musl = is_linux and abi == .musl`
  - `is_glibc = is_linux and !is_musl`
- Set `needs_libintl` for Windows, Linux musl, and macOS.
- Fail fast for unsupported operating systems instead of warning and continuing with Linux definitions.

#### 2. Split portable Unix and Linux-only sources

**File**: `build.zig`

- Keep these in the portable Unix source list:
  - `glib-unix.c`
  - `giounix.c`
  - `gspawn-posix.c`
  - `libcharset/localcharset.c`
- Move `gjournal-private.c` to a Linux-only list and add it only for `is_linux`.
- Do not compile `gosxutils.m`; `HAVE_COCOA` and `HAVE_CARBON` remain undefined.

#### 3. Generate Darwin-correct `glibconfig.h`

**File**: `build.zig`

Add an Apple branch for values that differ from Linux while retaining the shared LP64 values:

- `gint64`/`guint64`: `long long`/`unsigned long long`
- constants: `LL`/`ULL`
- format modifier: `ll`
- `gsize`, `gssize`, and pointer-sized types: `long`/`unsigned long`
- `G_OS_UNIX`
- `G_THREADS_IMPL_POSIX`
- `G_MODULE_SUFFIX`: `so`
- `GLIB_SYSDEF_AF_INET6`: 30
- existing poll, message, path, and byte-order constants remain unchanged

Keep the platform branches narrow so Linux and Windows generated output is unchanged.

#### 4. Replace broad non-Windows feature blocks

**File**: `build.zig`

Organize `config.h` into common POSIX, Linux-only, and Darwin-only sections.

Enable on Darwin:

- `BROKEN_POLL`
- `HAVE_CRT_EXTERNS_H`
- `HAVE__NSGETENVIRON`
- `HAVE_MACH_MACH_TIME_H`
- `HAVE_CLOCK_GETTIME`
- `HAVE_PTHREAD_ATTR_SETSTACKSIZE`
- `HAVE_PTHREAD_ATTR_SETINHERITSCHED`
- `HAVE_PTHREAD_COND_TIMEDWAIT_RELATIVE_NP`
- `HAVE_PTHREAD_SETNAME_NP_WITHOUT_TID`
- `HAVE_PTHREAD_GETNAME_NP`
- `HAVE_STRUCT_TM_TM_GMTOFF`
- `HAVE_UINT128_T`
- the verified portable POSIX headers and functions already used by the core sources

Do not enable on Darwin:

- `HAVE_SYS_PRCTL_H`
- `HAVE_SYS_AUXV_H`
- `HAVE_PPOLL`
- `HAVE_GETAUXVAL`
- `HAVE_GETRESUID`
- `HAVE_EVENTFD`
- `HAVE_SOCK_CLOEXEC`
- `HAVE_PTHREAD_CONDATTR_SETCLOCK`
- `HAVE_PTHREAD_SETNAME_NP_WITH_TID`
- `HAVE_CLOCK_NANOSLEEP`
- `HAVE_PTHREAD_GETAFFINITY_NP`

Keep `HAVE_O_CLOEXEC`, `HAVE_POSIX_SPAWN`, `HAVE_SPAWN_H`, `HAVE_MEMMEM`, and the other verified portable APIs enabled.

#### 5. Build and propagate Darwin intl/iconv dependencies

**File**: `build.zig`

- Include macOS in `needs_libintl`.
- Rename the curated gettext source list so its name no longer implies Windows-only use.
- Reuse the corrected pinned gettext `config.h`.
- Compile `setlocale-compat.c` into `intl_lib` for Windows or macOS.
- Install `libintl.a` and `libintl.h` on macOS through the existing artifact/header path.
- Link `intl_lib` into the GLib module.
- Link the macOS SDK iconv library:

```zig
if (is_macos)
    mod.linkSystemLibrary("iconv", .{});
```

- Continue using the vendored libiconv implementation only on Windows.
- Do not add Homebrew include or library paths.

### Success Criteria

- Native macOS compilation produces no Linux-header or Linux-API errors.
- `libglib-2.0.a` and `libintl.a` are installed.
- A downstream Zig executable that only links `glib_dep.artifact("glib-2.0")` resolves intl and iconv transitively.
- Darwin's public `glibconfig.h` reports `AF_INET6 == 30` and a `gint64` type compatible with `int64_t`.
- Existing Linux and Windows generated configuration remains unchanged except for intentional target-classification cleanup.

**Implementation Note**: Pause after the native GLib artifact and direct smoke test pass before adding the external consumer test.

---

## Phase 3: Add Static Smoke Tests and Native macOS CI

### Overview

Make the Darwin support reproducible and prove the installed artifact works without Homebrew GLib/gettext.

### Changes Required

#### 1. Add a `zig build test` smoke executable

**File**: `build.zig`

Follow the inline generated-C pattern already used by `cataggar/libslirp`.

The smoke test should:

- include the installed-style `<glib.h>` headers;
- assert `GLIB_SYSDEF_AF_INET6 == AF_INET6`;
- assert `gint64` is type-compatible with `int64_t`;
- perform a UTF-8 conversion with `g_convert()` to exercise iconv;
- call `g_get_monotonic_time_ns()` to exercise the Mach clock path;
- create and join a named `GThread` to exercise one-argument `pthread_setname_np`;
- perform a short `GCond` timed wait to exercise `pthread_cond_timedwait_relative_np`;
- call `g_setenv()`/`g_getenv()` and `g_spawn_sync()` to exercise Darwin environment and spawn handling;
- create and wake a `GMainContext` to cover the pipe fallback used without eventfd.

Link the smoke module only through `mod.linkLibrary(lib)`. Do not manually link intl or iconv in the test; successful linking is the transitive-dependency assertion.

#### 2. Add a Zig CI workflow

**File**: `.github/workflows/zig.yml`

- Use pinned `actions/checkout` and the existing repository convention from QEMU:

```yaml
- uses: cataggar/ghr/actions/install@v0.6.6
  with:
    tools: |
      cataggar/zig@v0.16.0 RWSGOq2NVecA2UPNdBUZykf1CCb147pkmdtYxgb3Ti+JO/wCYvhbAb/U
```

- Add these jobs or matrix entries:
  - `macos-latest`, native arm64: build, install, and run `zig build test`
  - `ubuntu-latest`, native glibc: build and run `zig build test`
  - `ubuntu-latest`, `x86_64-linux-musl`: build/install
  - `ubuntu-latest`, `x86_64-windows-gnu`: build/install
- Do not install Homebrew GLib, gettext, or libiconv in the macOS job.
- Assert the macOS install contains both archives.
- Use `otool -L` on the smoke executable to reject `/opt/homebrew` paths and any gettext dylib. The system `/usr/lib/libiconv` and `libSystem` are expected.
- Use `nm -u` or equivalent to reject Linux-only unresolved symbols such as `eventfd`, `getauxval`, `pthread_condattr_setclock`, `clock_nanosleep`, and `pthread_getaffinity_np`.

#### 3. Add libslirp's required macOS resolver linkage

**Repository**: `cataggar/libslirp`  
**File**: `build.zig`

- Add explicit macOS target detection.
- Link the macOS resolver library through the libslirp module:

```zig
if (is_macos)
    mod.linkSystemLibrary("resolv", .{});
```

The existing macOS source path calls the versioned resolver symbols
`res_9_ninit`, `res_9_getservers`, and `res_9_ndestroy`; these are not
provided by `libSystem` without `-lresolv`.

#### 4. Test the pinned libslirp consumer against the checkout

**File**: `.github/workflows/zig.yml`

- Check out the immutable current `cataggar/libslirp` Zig commit (`02e78324f0a57a705a641ae013b20f2044809d6f`) into a sibling directory.
- Override only its `glib` dependency with the workflow's current GLib checkout:

```sh
cd libslirp
zig fetch --save=glib "$GITHUB_WORKSPACE/glib"
zig build test -Dtarget=native -Doptimize=ReleaseFast
```

- Run this on native macOS. The existing smoke test calls `slirp_new()` and proves the full GLib -> libslirp static link.
- Use the exact libslirp commit rather than the moving `zig16` branch so CI remains deterministic.

#### 5. Update the libslirp pin after GLib lands

**Repository**: `cataggar/libslirp`  
**File**: `build.zig.zon`

- Update the pinned GLib URL/hash to the completed GLib commit.
- Run the libslirp native macOS smoke test without a local override.
- Keep the GLib workflow's local override test so future GLib pull requests continue testing the current checkout before a new immutable pin exists.

### Success Criteria

- The native macOS workflow passes without Homebrew target libraries.
- The GLib smoke test exercises Darwin-specific runtime branches.
- The pinned libslirp source builds and its `slirp_new()` smoke test passes against the current GLib checkout.
- CI remains green for native glibc, musl, and Windows/mingw builds.

---

## Phase 4: Regression Validation and Rollout

### Overview

Verify the supported target matrix and update downstream pins in dependency order.

### Validation Commands

Native macOS:

```sh
zig build -Dtarget=native -Doptimize=ReleaseFast --prefix zig-out-macos
zig build test -Dtarget=native -Doptimize=ReleaseFast
```

Linux glibc:

```sh
zig build -Dtarget=x86_64-linux-gnu -Doptimize=ReleaseFast --prefix zig-out-linux-glibc
```

Linux musl:

```sh
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseFast --prefix zig-out-linux-musl
```

Windows/mingw:

```sh
zig build -Dtarget=x86_64-windows-gnu -Doptimize=ReleaseFast --prefix zig-out-windows
```

Pinned libslirp:

```sh
zig fetch --save=glib /absolute/path/to/glib
zig build test -Dtarget=native -Doptimize=ReleaseFast
```

### Rollout Order

1. Land the Darwin-safe gettext commit.
2. Update the GLib gettext pin and land GLib issue #1.
3. Update the libslirp GLib pin and verify its native macOS test.
4. Let `cataggar/qemu#34` add its selective macOS libslirp dependency step and corresponding pkg-config metadata.
5. Address optional Cocoa/Carbon integration independently in GLib issue #2.

### Success Criteria

- All four supported target builds pass with Zig 0.16.
- Native macOS GLib and libslirp runtime smoke tests pass.
- No generated artifact or runtime load command references Homebrew GLib/gettext paths.
- The downstream pin chain is immutable and reproducible.

---

## Testing Strategy

### Build-Time Tests

- Compile each supported target with `ReleaseFast`.
- Check expected archive and header installation.
- Reject accidental Linux source/API leakage in the Darwin artifact.
- Check Darwin `glibconfig.h` ABI constants.

### Runtime Tests

- Run the GLib core smoke executable on native Linux glibc and macOS.
- Run the libslirp `slirp_new()` smoke executable on native macOS.
- Exercise iconv, intl/setlocale, pthread condition timing, thread naming, environment access, spawning, Mach monotonic time, and main-context wakeups.

### Static-Link Tests

- Link the GLib smoke test through the GLib artifact without manually naming intl or iconv.
- Confirm `libintl.a` is installed.
- Confirm the final macOS executable uses only Apple system dylibs and no Homebrew gettext/GLib dylibs.

## Performance Considerations

- The macOS artifact gains a separate static libintl archive but does not absorb optional Cocoa/Carbon code.
- Disabling eventfd and ppoll uses GLib's established portable pipe/select fallbacks on macOS.
- `mach_absolute_time()` preserves the upstream high-resolution monotonic clock path.

## Migration Notes

- There is no data migration.
- Dependency pins must be updated in order because Zig package URLs and hashes are immutable.
- QEMU's generated pkg-config metadata must eventually include `-lintl -liconv` for installed macOS static archives; the raw `.a` files cannot encode those flags themselves.

## References

- Original issue: https://github.com/cataggar/glib/issues/1
- Optional Cocoa/Carbon follow-up: https://github.com/cataggar/glib/issues/2
- Linked QEMU issue: https://github.com/cataggar/qemu/issues/34
- Homebrew GLib formula: https://github.com/Homebrew/homebrew-core/blob/e7707b2defd28aff78e708f18d711ba96d3a691b/Formula/g/glib.rb
- Homebrew static-library restoration: https://github.com/Homebrew/homebrew-core/commit/3690694f57d52bc5b3cbd2160f0a7b6c579f988e
- Homebrew gettext formula: https://github.com/Homebrew/homebrew-core/blob/e7707b2defd28aff78e708f18d711ba96d3a691b/Formula/g/gettext.rb
- Upstream Darwin source selection: `glib/meson.build:376-405`
- Upstream Darwin platform probes: `meson.build:1002-1127`
- Upstream integer ABI selection: `meson.build:1643-1776`
- Upstream socket constants: `meson.build:2036-2063`
- Upstream pthread probes: `meson.build:2129-2202`
- Existing libslirp consumer: https://github.com/cataggar/libslirp/tree/02e78324f0a57a705a641ae013b20f2044809d6f
