# Assumptions and Specifications (Included Packages / UID / Paths)

This document summarizes the **assumptions and specifications** for the Flutter development image built in this repository.  
For the “pull/FROM and use it” workflow, see `USAGE_en.md`. For build and maintenance details, see `README.md`.

## Base Image and User

- Base image: `mcr.microsoft.com/devcontainers/base:ubuntu`
- Default user: `vscode`
- Recommended operation: run with an **arbitrary UID** by specifying `user: <uid>:<gid>` in compose or `docker run`
  - If the UID/GID differs from the host, permission issues are more likely on bind mounts

## Main Paths and Permissions

- Flutter SDK: `FLUTTER_HOME=/opt/flutter`
  - The SDK itself is basically **read-only** (`root:root` / `a+rX`)
  - Exception: `/opt/flutter/bin/cache` is `1777` so it can be updated even with an arbitrary UID
- Pub cache:
  - Image default: `PUB_CACHE=/tmp/.pub-cache` (`1777`)
  - Recommended: in real use, point `PUB_CACHE` to a persistent volume under the project, for example `/workspace/.cache/pub-cache`
- HOME:
  - Recommended: point `HOME` to a persistent volume under the project, for example `/workspace/.home`

## Android-Bundled Image (`Dockerfile.android`)

### Component Overview

| Area | Web image | Android image |
| --- | --- | --- |
| Flutter SDK | Included | Included |
| Web support | Enabled through build arguments | Enabled through build arguments |
| Android SDK | Not included | Included |
| Emulator | Not included | Included |
| Typical use case | Flutter web development | Flutter web development plus Android emulator/device testing |

### Runtime Behavior

| Item | Behavior |
| --- | --- |
| Android SDK distribution location | `/opt/android-sdk.dist` |
| Runtime SDK cache | At startup, `/opt/android-sdk.dist` is initialized into `$HOME/.cache/android-sdk` |
| Environment variables | `ANDROID_SDK_ROOT`, `ANDROID_HOME`, and `PATH` are overwritten by `ENTRYPOINT` |
| Default AVD | `dev` is created during the build |

## iOS Image (`Dockerfile.ios`)

### Component Overview

| Area | Web image | iOS image |
| --- | --- | --- |
| Flutter SDK | Included | Included |
| Web support | Enabled through build arguments | Enabled through build arguments |
| iOS support | Not included | Included |
| Xcode | Not included | Not included |
| Typical use case | Flutter web development | Flutter web development plus preparation of Flutter artifacts for iOS |

### Runtime Behavior

| Item | Behavior |
| --- | --- |
| iOS Flutter artifacts | Pre-fetched with `flutter precache --ios` |
| Build/signing | Not complete on Ubuntu containers; actual iOS device builds and signing require a macOS environment with Xcode |

## Included Packages

### Web Image (`Dockerfile`)

| Package | Purpose |
| --- | --- |
| `curl` | Fetching Flutter and other external downloads |
| `git` | Cloning the Flutter SDK and running `git config --global` |
| `unzip` | Extracting Flutter-related archives |
| `xz-utils` | Compression archive support |
| `zip` | Compression archive support |
| `libglu1-mesa` | Flutter/GUI runtime support |
| `openjdk-17-jdk` | Java runtime for Flutter/Android-related tooling |

### Android Image (`Dockerfile.android`)

| Package | Purpose |
| --- | --- |
| `ca-certificates` | HTTPS certificate validation |
| `curl` | Fetching Flutter and Android cmdline-tools |
| `git` | Cloning the Flutter SDK and running `git config --global` |
| `unzip` | Extracting Flutter and Android cmdline-tools archives |
| `xz-utils` | Compression archive support |
| `zip` | Compression archive support |
| `libglu1-mesa` | Flutter/GUI runtime support |
| `openjdk-17-jdk` | Java runtime for Flutter and Android tooling |
| `libnss3` | Android Emulator runtime dependency |
| `libx11-6` | Android Emulator X11 dependency |
| `libxcomposite1` | Android Emulator rendering dependency |
| `libxcursor1` | Android Emulator rendering dependency |
| `libxdamage1` | Android Emulator rendering dependency |
| `libxext6` | Android Emulator X11 extension dependency |
| `libxfixes3` | Android Emulator rendering dependency |
| `libxi6` | Android Emulator input dependency |
| `libxrandr2` | Android Emulator screen management dependency |
| `libxrender1` | Android Emulator rendering dependency |
| `libxtst6` | Android Emulator input/testing dependency |
| `libpulse0` | Android Emulator audio dependency |
| `libdrm2` | Android Emulator rendering dependency |
| `libxkbcommon0` | Android Emulator keyboard dependency |
| `libasound2t64` or `libasound2` | Android Emulator audio dependency; `libasound2t64` is preferred when available, otherwise `libasound2` is installed |

### Android-Side Extra Components

| Component | Details |
| --- | --- |
| Android cmdline-tools | Tooling that includes `sdkmanager` and `avdmanager` |
| Android platform-tools | Includes `adb` and related tools |
| Android platform | `platforms;android-34` |
| Android build-tools | `build-tools;34.0.0` |
| Android emulator | The `emulator` package |
| Android system image | `system-images;android-34;google_apis;x86_64` |

## Compatibility and Constraints

- Rootless execution is assumed, with Podman in mind.
- Emulator hardware acceleration depends on the host and requires KVM.
- Direct USB access to a real device is highly host-dependent, so the recommended approach is to use the host `adb` plus wireless debugging over TCP.
- iOS device builds and signing cannot be completed inside this Linux container alone; a macOS environment with Xcode is required.
