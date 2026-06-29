# How to Use the Container Image Only (FROM / pull)

This document is for people who want to **use** `ubuntu-flutter-container` as a base image, either by `FROM`-ing it or by pulling it and building on top of it.  
For assumptions about usage such as UID, paths, and included packages, see `IMAGE_NOTES_en.md`. For the build process and design intent of this repository itself, see `README.md`.

## Intended Usage

- Prepare a `Dockerfile` in a downstream project and use this image via `FROM`
- Handle runtime UID/GID, `HOME`, and `PUB_CACHE` on the downstream side with compose or `docker run` options

## Minimal Downstream Example (Web / Common)

### Dockerfile (downstream side)

Replace the image name with one that matches your environment, such as `localhost/...` for local use or `ghcr.io/<org>/...` for a registry.

```Dockerfile
FROM localhost/flutter-base-anyuid:latest
WORKDIR /workspace
CMD ["/bin/bash"]
```

### docker-compose.yml (downstream side)

Arbitrary UID handling and cache persistence are absorbed by compose in the downstream project.

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: ./Dockerfile
    working_dir: /workspace
    network_mode: host
    tty: true
    stdin_open: true

    user: "${UID:-1000}:${GID:-1000}"

    environment:
      - HOME=/workspace/.home
      - PUB_CACHE=/workspace/.cache/pub-cache

    volumes:
      - .:/workspace:cached
      - ./.cache/home:/workspace/.home
      - ./.cache/pub-cache:/workspace/.cache/pub-cache
```

Start example:

```bash
export UID
export GID="$(id -g)"
podman compose up --build
```

## Web + Android (Up to Emulator Execution)

This repository includes `Dockerfile.android` as a derivative image that contains the Android SDK and Emulator, separate from the lightweight Web image.

- Web image: `Dockerfile`
- Web + Android image: `Dockerfile.android`

The basic rule is to choose which one to `FROM` in the downstream project.

### Emulator Prerequisites on the Host

- A **Linux host with KVM available** (`/dev/kvm` must exist)
- The user running the container must be able to access **`/dev/kvm`**; in many environments this means being in the `kvm` group
- If you need the emulator GUI, **X11 forwarding** is required, and the exact setup depends on your environment

### Emulator Start Example (Inside the Container)

```bash
emulator -list-avds
emulator -avd dev
```

Headless mode:

```bash
emulator -avd dev -no-window -gpu swiftshader_indirect
```

Note:

- The Android SDK is initialized from `/opt/android-sdk.dist` into `$HOME/.cache/android-sdk` at startup. If you persist `HOME`, the cache will persist too.

## Web + iOS (Preparing the Flutter SDK)

This repository also includes `Dockerfile.ios` as a derivative image for **iOS + Web** in addition to the lightweight Web image.

- Web image: `Dockerfile`
- Web + iOS image: `Dockerfile.ios`

The basic rule is to choose which one to `FROM` in the downstream project.

### Usage Hint

```bash
podman compose -f docker-compose.ios.yml up --build
```

Note:

- On Ubuntu containers, you can prepare Flutter artifacts for iOS, but **actual iOS device builds and signing require a macOS environment with Xcode**.
- This image is intended to prepare a Flutter SDK for iOS + Web workflows.

## Verifying on a Physical Android Device Without Adding More docker/compose Complexity

Direct USB wiring into the container (`/dev/bus/usb` / `privileged`) is highly host-dependent, so for physical devices it is lighter to rely on **host `adb` plus wireless debugging over TCP**.

Prerequisites:

- The phone and host are on the same network, or otherwise reachable
- `adb` is available on the host, for example via `android-tools-adb` on Linux

### Procedure for Android 11+ "Wireless debugging"

```bash
adb pair <phone-ip>:<pair-port>
adb connect <phone-ip>:<connect-port>
adb devices
```

Inside the container, if needed:

```bash
adb devices
flutter devices
flutter run
```

### Older Method (Switch to TCP After a One-Time USB Connection)

```bash
adb tcpip 5555
adb connect <phone-ip>:5555
```
