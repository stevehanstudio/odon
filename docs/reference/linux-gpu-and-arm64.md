# Linux, GPU, and ARM64

This page summarizes how `odon` uses OpenGL on Linux, why behavior can differ between **macOS** and **Linux ARM64**, and what to check when the UI or viewer features seem broken.

## Stack

`odon` is built with **Rust**, **`eframe`**, **`egui_glow`**, and **`glow`**. The default `eframe` features used by this project include **`glow`** (OpenGL), plus **`wayland`** and **`x11`** for native windowing on Linux.

Rendering and many data paths assume a working **OpenGL context**.

## What `gpu_available` means in `odon`

Inside the app, **`gpu_available` is set from `cc.gl.is_some()`** in the `eframe` creation context:

- If **`cc.gl` is `Some`**, an OpenGL context was created and GPU-accelerated paths (tile compositing, many overlays, mosaic, etc.) can run.
- If **`cc.gl` is `None`**, the window may still appear, but **large parts of the viewer are disabled or degraded**.

So “GUI issues” on Linux are often really **missing or unstable OpenGL**, not generic UI bugs.

## Features that depend on the GPU path

Approximate behavior:

- **Mosaic mode** refuses to start without a GPU/OpenGL backend (explicit error in the app).
- **SpatialData** and other workflows may show status text when secondary image layers need the GPU path.
- **Overlays, points, labels, threshold preview**, and similar features are tied to the same GPU flag.

See also [Current limitations](../advanced/current-limitations.md) (mosaic depends on the GPU rendering path).

## Build and runtime dependencies (Linux)

CI installs one package on Ubuntu before `cargo check`:

- `libgtk-3-dev` — windowing and **native file dialogs** (`rfd` on Linux needs GTK).

On **Debian/Ubuntu**-style systems, install the same or equivalent runtime package (the `-dev` name maps to `libgtk-3-0` for running a prebuilt binary). Other distros (Fedora, Arch, NixOS) need the analogous GTK3 and X11/Wayland development packages for **building from source**.

OpenGL itself is loaded at runtime rather than linked at build time, so no GL `-dev` package is required to compile. You do need a working GL driver to run.

## Known-good configuration

The following ARM64 setup builds and runs the full GPU path, including mosaic mode:

| Component | Value |
| --- | --- |
| Hardware | NVIDIA GB10 (aarch64) |
| Driver | NVIDIA proprietary 580.173.02 |
| Session | X11 |
| GLX vendor | `NVIDIA Corporation` (both client and server) |
| Reported GL | `3.3.0 NVIDIA 580.173.02`, GLSL `3.30` |
| Toolchain | rustc 1.94.1, `aarch64-unknown-linux-gnu` |

The reported GL and GLSL versions matter: the renderer's primary shaders are `#version 330 core`, so a stack reporting GL 3.3 or later on the **desktop** profile is what you want. See [Known limitations on ARM64](#known-limitations-on-arm64) below.

## Troubleshooting (especially Linux ARM64)

1. **Start with Odon’s own startup log**  
   Run with `--log-level info` and look for the `startup:` lines, which are the fastest way to classify the problem:

   ```bash
   odon --log-level info
   ```

   ```
   startup: os=linux arch=aarch64 opengl_context=true
   startup: XDG_SESSION_TYPE=x11
   startup: DISPLAY=:1
   startup: gl_vendor="NVIDIA Corporation" gl_renderer="NVIDIA GB10/PCIe" gl_version="3.3.0 NVIDIA 580.173.02" glsl_version="3.30 NVIDIA via Cg compiler"
   ```

   Read them as follows:

   - `opengl_context=false` — no GL context at all. The app warns and points here; everything below applies.
   - `gl_renderer` containing **`llvmpipe`**, **`softpipe`**, or **`swrast`** — you are on **software rendering**. The app will work but be very slow, and this usually means the GPU driver is not being picked up by your session.
   - `gl_version` below **3.3** — the renderer falls back to its GLSL 120 shader path and some overlays are unavailable.

2. **Confirm OpenGL from the OS**  
   Use your distro’s tools (e.g. `glxinfo` on X11, or `eglinfo` where appropriate) to verify that a sensible GL stack is available. On ARM64, drivers and compositor choices vary more than on a typical macOS install.

3. **Try forcing X11 for winit** (when you use an X session or XWayland is acceptable):

   ```bash
   WINIT_UNIX_BACKEND=x11 odon
   ```
   
   Or with a local build:
   
   ```bash
   WINIT_UNIX_BACKEND=x11 cargo run -- --log-level info
   ```

4. **Compare Wayland vs X11 session**  
   Some stacks create GL contexts reliably in one mode but not the other (especially with certain GPU drivers). The known-good configuration above uses X11.

5. **Software GL (last resort for diagnosis)**  
   Setting `LIBGL_ALWAYS_SOFTWARE=1` can help determine whether the problem is driver-specific (performance will be poor; use only for testing). The startup log echoes this variable when it is set, so a forgotten export is easy to spot.

6. **Verbose app logging**  
   If the `startup:` lines look healthy but a specific feature misbehaves, raise the level to capture the data and rendering paths:

   ```bash
   odon --log-level debug
   ```

## Known limitations on ARM64

The renderer's shaders target **desktop OpenGL** only: `#version 330 core`, with a `#version 120` fallback for older contexts. There is no `#version 300 es` variant.

In practice this means a device that exposes **only OpenGL ES** (with no desktop GL profile) will create a context but fail to compile shaders. ARM64 systems using the NVIDIA proprietary driver expose desktop GL through GLX and are unaffected. If you hit this on an EGL/GLES-only board, that is a genuine gap rather than a configuration problem.

## CI and release coverage

- **Linux CI** covers both **x86_64** (`ubuntu-latest`) and **arm64** (`ubuntu-24.04-arm`), so arm-native build breaks are caught early. Note that GitHub's arm64 runners are free for public repositories; a private fork may need a self-hosted arm64 runner instead.
- **GitHub Releases** ship a Debian package per Linux architecture: **`odon_<version>_amd64.deb`** and **`odon_<version>_arm64.deb`**. If your machine is ARM64, install the **arm64** package or **build from source** on the device.

## Related documentation

- [CLI reference](cli.md)
- [Mosaic workflow](../workflows/mosaic.md)
- [SpatialData and Xenium](../data-sources/spatialdata-and-xenium.md)
