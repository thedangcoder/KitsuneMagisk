# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**KitsuneMagisk** is a fork of Magisk, a root solution for Android. This fork is currently archived (as of August 2025) but maintained for historical and educational purposes.

### Key Differences from Upstream Magisk
- **Zygisk Removed**: Zygisk code injection mechanism has been completely removed
- **MagiskHide Focused**: Emphasis on maintaining MagiskHide functionality to hide root from apps
- **SELinux Compatibility**: Enhanced support for SELinux-disabled environments (Waydroid, etc.)
- Commits show specific fixes for `is_zygote()` to handle environments without SELinux

## Build Commands

### Environment Setup
```bash
# Install NDK (custom ONDK for this project)
./build.py ndk

# Set environment variables (required)
export ANDROID_SDK_ROOT=/path/to/android/sdk
export ANDROID_STUDIO=/path/to/android/studio  # Optional but recommended for JDK
```

### Building
```bash
# Build everything (binaries + APK)
./build.py all

# Build only native binaries
./build.py binary

# Build specific targets (magisk, magiskinit, magiskboot, magiskpolicy, busybox, resetprop)
./build.py binary magisk magiskpolicy

# Build only the Android app
./build.py app

# Build stub app
./build.py stub

# Release builds (add -r flag)
./build.py -r all

# Clean build artifacts
./build.py clean
# Clean specific components: native, cpp, rust, java
./build.py clean native
```

### Testing
```bash
# Run emulator tests (requires AVD setup)
# Tests API levels: 23, 26, 28, 29, 35
./scripts/avd_test.sh

# Test specific API level
./scripts/avd_test.sh 28
```

### Configuration
- Build config: `config.prop` (see `config.prop.sample` for defaults)
- Version info is auto-generated from git commit hash if not specified
- Default output directory: `out/`

## Architecture

### High-Level Structure
```
native/src/          # Native code (C++ + Rust mixed)
├── base/           # Rust base utilities
├── boot/           # Boot image manipulation (magiskboot)
├── core/           # Core Magisk functionality
│   └── deny/       # MagiskHide implementation
├── init/           # MagiskInit (early boot)
├── sepolicy/       # SELinux policy manipulation (magiskpolicy)
└── external/       # External dependencies (cxx-rs)

app/                # Android application (Kotlin/Java)
├── src/main/
│   ├── java/       # Java sources
│   └── aidl/       # AIDL interfaces for IPC
└── shared/         # Shared code between app and stub

stub/               # Stub application (hidden app)
```

### Key Components

**Native Binaries** (`native/src/`)
- **magisk**: Main root daemon and CLI (Rust + C++)
- **magiskinit**: Replaces `init`, injects Magisk into boot process (C++)
- **magiskboot**: Boot image patching tool (Rust)
- **magiskpolicy**: SELinux policy patching (C++)
- **busybox**: Busybox implementation
- **resetprop**: System property manipulation bypassing `property_service`

**MagiskHide** (`native/src/core/deny/`)
- `cli.cpp`: Command-line interface for MagiskHide
- `ptrace.cpp`: Ptrace-based process monitoring/hiding
- `revert.cpp`: Revert hiding operations
- `utils.cpp`: Utility functions for process detection

**Build System**
- `build.py`: Main build script (Python)
  - Handles native builds via NDK and Cargo
  - Manages APK builds via Gradle
  - Creates generated headers in `native/out/generated/`
- Gradle with custom `MagiskPlugin` for Android components

### Boot Process
1. **Pre-Init**: `magiskinit` replaces `/init`, mounts partitions, patches `init.rc`, loads/patches SELinux policies
2. **post-fs-data**: `magiskd` daemon starts, executes scripts, magic mounts modules
3. **late_start**: Service mode starts, executes service scripts

### Data Paths
- **Magisk tmpfs**: `/sbin` or `/debug_ramdisk` (check with `magisk --path`)
- **Secure dir**: `/data/adb/` (modules, scripts, database)
- **Modules**: `/data/adb/modules/` and `/data/adb/modules_update/`

## Language Breakdown

| Component | Languages | Notes |
|-----------|-----------|-------|
| Native binaries | Rust + C++ | Mixed via CXX bridge |
| Android app | Kotlin + Java | AIDL for IPC |
| Build system | Python + Gradle | Custom build.py script |
| SELinux | C++ | libsepol wrapper |

## Development Notes

### Working with Native Code
- Run `./build.py binary` before editing native code - generates required headers
- Rust workspace is in `native/src/` with 5 crates: base, boot, core, init, sepolicy
- C++ uses NDK build with `Android.mk`
- CXX bridge used for Rust/C++ interop (see `external/cxx-rs`)

### Android Development
- Open in Android Studio as a project
- Kotlin/Java/C++ should work out-of-the-box
- For Rust support: link ONDK toolchain via rustup, install IntelliJ Rust plugin

### Important Files
- `config.prop`: Build configuration (version, signing keys)
- `build.py`: Entry point for all builds
- `native/src/core/bootstages.cpp`: Boot stage logic
- `native/src/core/deny/`: MagiskHide implementation
- `docs/build.md`: Detailed build instructions
- `docs/details.md`: Internal implementation details

### Recent Fork Changes
- Commit `c30ba784c`: Fixed `is_zygote()` for SELinux-disabled environments
- Commit `2ef8f002a`: Removed Zygisk completely
- Commit `11bd02d35`: Removed API 26 from tests
- No Zygisk means this fork focuses purely on traditional root methods + hiding

## Security Context

This is a defensive security tool (root hiding) licensed under GPL-3.0-or-later. When working with this codebase:
- Focus on understanding MagiskHide's process detection/hiding mechanisms
- The `deny/` directory implements hiding via ptrace and process monitoring
- SELinux policy patches in `sepolicy/` are for root functionality, not exploitation
