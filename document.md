# scrcpy — Repository Document

> **Version:** 5.0.1 · **License:** Apache 2.0 · **Upstream:** <https://github.com/Genymobile/scrcpy>
>
> This document is a map of the repository: what the project is, how it is
> organized, how to build and test it, and where to look for more detail.
> For user-facing documentation, see [README.md](README.md) and the
> [`doc/`](doc/) folder.

---

## 1. Overview

**scrcpy** (pronounced "*screen copy*") mirrors an Android device's screen
(and optionally its audio) on a computer and lets you control it with the
computer's keyboard and mouse. It connects over USB or TCP/IP, requires no root
access, and does not install any app on the device. It runs on Linux, Windows,
and macOS.

Key characteristics:

- **Lightweight:** native client, shows only the device screen.
- **Low latency:** roughly 35–70 ms.
- **Non-intrusive:** the server is pushed to the device temporarily and nothing
  stays installed.
- **Feature-rich:** audio forwarding (Android 11+), recording, virtual displays,
  camera mirroring (Android 12+), V4L2 webcam output (Linux), HID keyboard and
  mouse simulation, gamepad support, OTG mode, and copy-paste in both directions.

---

## 2. Architecture

scrcpy is split into two programs that talk over the network:

| Part | Language | Runs on | Location | Output |
|------|----------|---------|----------|--------|
| **Client** (`scrcpy`) | C (C11) + SDL/FFmpeg | Host computer | [`app/`](app/) | Executable |
| **Server** (`scrcpy-server`) | Java (Android) | Android device | [`server/`](server/) | `scrcpy-server` JAR, pushed to `/data/local/tmp/scrcpy-server.jar` |

The client pushes the server to the device with ADB, starts it, and then
exchanges data over up to three sockets:

| Socket | Direction | Content |
|--------|-----------|---------|
| Video | device → client | Encoded screen stream (H.264 by default, also H.265) with per-packet headers |
| Audio | device → client | Encoded audio stream (Opus by default) — or microphone with `--audio-source=mic` |
| Control | bidirectional | Input events client → device; clipboard changes device → client |

The server sends the device name first (used as the window title). Each socket
is handled by a dedicated read/write thread on both sides. Any of video, audio,
or control may be disabled, but not all of them at once.

### Connection model

By default (without `--force-adb-forward`) the roles are inverted at the network
level: the client opens a listening socket and the server connects back to it.
This avoids race conditions without polling. See
[`doc/develop.md`](doc/develop.md) for the full protocol description.

### Data flow

```
 Host (client: scrcpy)                         Android device (server: scrcpy-server)
 ┌──────────────────────────┐                  ┌───────────────────────────────┐
 │ CLI / options            │   ADB push+start │ Screen capture / encoder      │
 │ ADB tunnel & receiver    │ ───────────────▶ │ Audio capture / encoder       │
 │ Demuxer → Decoder        │ ◀─── video ───── │ Input injection (control)     │
 │ Screen / SDL window      │ ◀─── audio ───── │ Clipboard monitoring          │
 │ Input manager → controls │ ──── control ──▶ │                               │
 │ Recorder / V4L2 sink     │                  │                               │
 └──────────────────────────┘                  └───────────────────────────────┘
```

---

## 3. Repository Layout

```
scrcpy/
├── app/                    Client (C)
│   ├── src/                Client source code (see §4)
│   ├── tests/              Unit tests for the client (C)
│   ├── deps/               Scripts to fetch/build dependencies (FFmpeg, SDL, libusb, dav1d, ADB)
│   ├── data/               Icons, prebuilt resources
│   ├── meson.build         Client build definition
│   ├── scrcpy.1            Man page
│   └── scrcpy-windows.*    Windows resource/manifest files
├── server/                 Server (Java, Android)
│   ├── src/main/           Server source code (see §5)
│   ├── src/test/           Server unit tests (JUnit)
│   ├── scripts/            Helper scripts
│   ├── build.gradle        Gradle/Android build
│   ├── meson.build         Meson wrapper that drives the Gradle build
│   ├── build_without_gradle.sh  Builds the server with javac/d8 only
│   └── proguard-rules.pro  ProGuard configuration
├── doc/                    User and developer documentation (Markdown)
├── release/                Release, packaging, and test scripts
├── config/                 Checkstyle configuration (Java) and related config
├── gradle/                 Gradle wrapper files
├── assets/                 Screenshots and other images
├── .github/                CI workflows, issue templates, funding file
├── meson.build             Top-level build definition
├── meson_options.txt       Build options (see §6.3)
├── build.gradle            Top-level Gradle build
├── settings.gradle
├── gradle.properties
├── gradlew / gradlew.bat   Gradle wrapper
├── run                     Helper script
├── install_release.sh      Installs a release build
├── bump_version            Updates the version number across files
├── cross_win32.txt         Meson cross file for 32-bit Windows (from Linux)
├── cross_win64.txt         Meson cross file for 64-bit Windows (from Linux)
├── README.md               Main project README
├── FAQ.md                  Frequently asked questions
└── LICENSE                 Apache License 2.0
```

---

## 4. Client (`app/`)

The client is written in C11. It uses **SDL** for windowing, input, and
rendering, **FFmpeg** (libavcodec) for decoding, and **libusb** for HID/OTG
features. Major modules in `app/src/`:

| Area | Files | Responsibility |
|------|-------|----------------|
| Entry point | `main.c`, `scrcpy.c/.h` | Startup, main loop, lifecycle |
| Options & CLI | `options.c/.h`, `cli.c/.h` | Parse and validate command-line options |
| ADB | `adb/` (`adb.c`, `adb_device.c`, `adb_parser.c`, `adb_tunnel.c`) | Talk to `adb`, list devices, set up forward/reverse tunnels |
| Server control | `server.c/.h` | Push the server JAR, start it, manage its process |
| Networking | `receiver.c/.h`, `demuxer.c/.h`, `device_msg.c/.h`, `control_msg.c/.h` | Socket reading, packet demuxing, message (de)serialization |
| Video | `decoder.c/.h`, `hwdec.c/.h`, `frame_buffer.c/.h`, `video_regulator.c/.h`, `vaapi_shim.c` | Decoding (software and hardware: VA-API, D3D11VA, VideoToolbox) |
| Audio | `audio_player.c/.h`, `audio_regulator.c/.h`, `util/audiobuf.c` | Audio playback with latency regulation |
| Rendering | `screen.c/.h`, `texture.c/.h`, `opengl.c/.h`, `egl.c/.h`, `icon.c/.h` | SDL window, textures, OpenGL/EGL, window icon |
| Input | `input_manager.c/.h`, `controller.c/.h`, `events.c/.h`, `mouse_capture.c/.h`, `keyboard_sdk.c/.h`, `mouse_sdk.c/.h`, `shortcut_mod.h` | Keyboard, mouse, and shortcuts; mapping to Android events |
| HID / OTG | `hid/` (`hid_keyboard.c`, `hid_mouse.c`, `hid_gamepad.c`), `uhid/`, `usb/` | Physical keyboard/mouse/gamepad simulation via UHID or OTG (libusb) |
| Recording | `recorder.c/.h` | MP4/MKV recording (`--record`) |
| Camera / webcam | `v4l2_sink.c/.h` | Expose the device as a V4L2 webcam (Linux) |
| Files | `file_pusher.c/.h` | Push files to the device by drag-and-drop |
| Platform | `compat.c/.h`, `sys/`, `android/`, `interop/`, `sdl_hints.c/.h` | OS-specific code and compatibility shims |
| Utilities | `util/` (`log.c`, `file.c`, `env.c`, `intmap.c`, `vector.h`, `strbuf.c`, `thread.h`, …) | Logging, strings, containers, threads, process helpers |
| Clock / stats | `clock.c/.h`, `fps_counter.c/.h`, `disconnect.c/.h` | Timing, FPS display, disconnection handling |

### Tests (`app/tests/`)

Small C unit tests covering ADB parsing, the audio buffer, binary helpers, CLI
parsing, Windows command-line handling, orientation, control-message
serialization, device-message deserialization, and the string/vector/deque
utilities. They are compiled with AddressSanitizer and UBSan by
`release/test_client.sh`.

---

## 5. Server (`server/`)

The server is a Java program executed on the device with `app_process` (started
through `adb shell`), not installed as an app. It uses hidden Android APIs to capture
the screen and audio, encode them, and inject input events. It is organized
under `com.genymobile.scrcpy`:

| Package | Responsibility |
|---------|----------------|
| `Server` | Entry point; argument parsing and session setup |
| `Options` | Server-side option handling (mirrors the client options) |
| `AndroidVersions`, `Workarounds` | API-level checks and per-device/per-version fixes |
| `AsyncProcessor`, `CleanUp` | Background tasks and cleanup on exit |
| `FakeContext` | Minimal Android context for running outside an app |
| `video/` | Screen capture and video encoding (MediaCodec) |
| `audio/` | Audio capture (`AudioRecord`/playback capture) and Opus/AAC/raw encoding |
| `control/` | Reading control messages, injecting input, clipboard handling, device-message writing |
| `device/` | Device info, orientation, and clipboard access |
| `display/` | Virtual displays and display management (`--new-display`) |
| `opengl/` | OpenGL-based frame processing (filters, crop, rotation) |
| `wrappers/` | Wrappers around hidden Android system services (`ServiceManager`, `DisplayManager`, `InputManager`, …) |
| `model/` | Data types (`Size`, `Rect`, `Point`, …) |
| `util/` | Logging, I/O, and helpers |

### Tests

JUnit tests under `server/src/test/java/com/genymobile/scrcpy/` cover the
protocol messages (`control/ControlMessageReaderTest`, `control/DeviceMessageWriterTest`),
codec options (`model/`), and utilities (`util/`: binary search, binary helpers,
command parsing, string utilities).

---

## 6. Building

Full instructions are in [`doc/build.md`](doc/build.md) and per-platform guides
([Linux](doc/linux.md), [Windows](doc/windows.md), [macOS](doc/macos.md)).
Summary:

### 6.1 Requirements

- **Client:** Meson ≥ 0.49, Ninja, a C11 compiler, pkg-config, and the client
  dependencies (SDL ≥ 3.2, libavcodec ≥ 60.3, libusb, and others depending on platform).
- **Server:** JDK, the Android SDK (set `ANDROID_SDK_ROOT`), and the Gradle
  wrapper (included). Not needed if you use a prebuilt server.
- **Device:** Android 5.0+ (API 21). Audio forwarding requires Android 11+
  (API 30). Camera mirroring requires Android 12+.

### 6.2 Quick build

```bash
# Full build (client + server) from sources
export ANDROID_SDK_ROOT=~/Android/Sdk      # adjust for your OS
meson setup x --buildtype=release --strip -Db_lto=true
ninja -Cx                                 # run as a non-root user

# Client only, using a prebuilt server
meson setup x -Dprebuilt_server=/path/to/scrcpy-server
ninja -Cx
```

On Windows, cross-compiling from Linux uses `cross_win32.txt` /
`cross_win64.txt`.

### 6.3 Build options (`meson_options.txt`)

| Option | Default | Purpose |
|--------|---------|---------|
| `compile_app` | `true` | Build the client |
| `compile_server` | `true` | Build the server |
| `prebuilt_server` | — | Path to a prebuilt `scrcpy-server` |
| `portable` | `false` | Look for `scrcpy-server` next to the executable |
| `static` | `false` | Link static dependencies |
| `server_debugger` | `false` | Wait for a debugger to attach to the server |
| `v4l2` | `true` | Enable V4L2 webcam support (Linux) |
| `usb` | `true` | Enable HID/OTG features |
| `vaapi` | `true` | VA-API hardware decoding (Linux) |
| `d3d11va` | `true` | D3D11VA hardware decoding (Windows) |
| `videotoolbox` | `true` | VideoToolbox hardware decoding (macOS) |

### 6.4 Dependencies

Helper scripts in `app/deps/` fetch and build the client's third-party
libraries: `ffmpeg.sh`, `sdl.sh`, `libusb.sh`, `dav1d.sh`, and platform-specific
ADB downloaders (`adb_linux.sh`, `adb_macos.sh`, `adb_windows.sh`).

---

## 7. Testing

| Target | Command | What it runs |
|--------|---------|--------------|
| Client unit tests | `release/test_client.sh` | Meson `test` target with ASan + UBSan (`app/tests/`) |
| Server unit tests | Gradle `test` task in `server/` | JUnit tests in `server/src/test/` |
| Lint (Java) | Checkstyle using `config/checkstyle/checkstyle.xml` | Style checks on server code |

The continuous-integration workflow lives in
[`.github/workflows/`](.github/workflows/). Release automation is in
[`release.yml`](.github/workflows/release.yml) and the scripts under
[`release/`](release/).

---

## 8. Releases and Packaging

- `bump_version` updates the version string across the project.
- `release/build_*.sh` build release artifacts for each platform.
- `release/package_client.sh` and `release/package_server.sh` create archives.
- `release/generate_checksums.sh` produces checksums for published files.
- `doc/verify-release.md` explains how users can verify downloads.
- `install_release.sh` installs a release build on the host.

Official releases are published only on the upstream GitHub repository.

---

## 9. Protocol

The client and server exchange binary messages. The protocol is documented
only through the unit tests on both sides:

- **Control messages (client → device):** serialization in
  [`app/tests/test_control_msg_serialize.c`](app/tests/test_control_msg_serialize.c),
  deserialization in
  [`ControlMessageReaderTest.java`](server/src/test/java/com/genymobile/scrcpy/control/ControlMessageReaderTest.java).
- **Device messages (device → client):** serialization in
  [`DeviceMessageWriterTest.java`](server/src/test/java/com/genymobile/scrcpy/control/DeviceMessageWriterTest.java),
  deserialization in
  [`test_device_msg_deserialize.c`](app/tests/test_device_msg_deserialize.c).

Start with [`doc/develop.md`](doc/develop.md) for the wider design.

---

## 10. Documentation Index (`doc/`)

| Topic | File |
|-------|------|
| Building from source | [build.md](doc/build.md) |
| Developer overview & protocol | [develop.md](doc/develop.md) |
| Linux / Windows / macOS install | [linux.md](doc/linux.md), [windows.md](doc/windows.md), [macos.md](doc/macos.md) |
| Connecting (USB / TCP/IP) | [connection.md](doc/connection.md) |
| Tunnels | [tunnels.md](doc/tunnels.md) |
| Video options | [video.md](doc/video.md) |
| Audio forwarding | [audio.md](doc/audio.md) |
| Device control & copy-paste | [control.md](doc/control.md) |
| Keyboard / mouse / gamepad | [keyboard.md](doc/keyboard.md), [mouse.md](doc/mouse.md), [gamepad.md](doc/gamepad.md) |
| Shortcuts | [shortcuts.md](doc/shortcuts.md) |
| Window behaviour | [window.md](doc/window.md) |
| Recording | [recording.md](doc/recording.md) |
| Camera mirroring | [camera.md](doc/camera.md) |
| Virtual displays | [virtual-display.md](doc/virtual-display.md) |
| V4L2 webcam (Linux) | [v4l2.md](doc/v4l2.md) |
| OTG mode | [otg.md](doc/otg.md) |
| Device options | [device.md](doc/device.md) |
| Verifying releases | [verify-release.md](doc/verify-release.md) |

Other project documents: [README.md](README.md), [FAQ.md](FAQ.md).

---

## 11. Contributing and Support

- **Bugs and feature requests:** <https://github.com/Genymobile/scrcpy/issues>
  (read the [FAQ](FAQ.md) first).
- **Community:** the project is discussed on Reddit (`r/scrcpy`) and social
  media, as listed in the README.
- **Maintainer:** Romain Vimont ([@rom1v](https://github.com/rom1v)).
- **Issue templates:** [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/).

---

## 12. License

scrcpy is free and open-source software, licensed under the
**Apache License, Version 2.0**. See [LICENSE](LICENSE).

Copyright © 2018 Genymobile; Copyright © 2018–2026 Romain Vimont.
