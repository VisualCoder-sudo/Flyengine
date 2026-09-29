# Flyengine

A 3D game engine and editor built on a modified
[raylib](https://www.raylib.com/) and [Box3D](https://github.com/erincatto/box3d).

Terrain with splatmap painting and LOD geomorphing, water bodies, rigid-body
physics, glTF/OBJ/FBX/PLY model import, a PBR terrain shader, an ImGui editor,
and a C# scripting host (Windows).

Builds from **one `CMakeLists.txt` on Linux and Windows**.

---

## Building

### Requirements

CMake 3.25 or newer, and a C++17 compiler (GCC 12+, Clang 15+, or MSVC 2022).

raylib, Box3D and a vendored libcurl are built from the tree as part of the
project, so there is nothing else to clone.

#### Linux

Debian/Ubuntu:

```sh
sudo apt install build-essential cmake ninja-build pkg-config \
    libasound2-dev libx11-dev libxrandr-dev libxi-dev libxcursor-dev \
    libxinerama-dev libxkbcommon-dev libgl1-mesa-dev libglib2.0-dev
# Optional, but strongly recommended -- see "Shader validation" below.
sudo apt install glslang-tools
```

Arch:

```sh
sudo pacman -S base-devel cmake ninja pkgconf \
    alsa-lib libx11 libxrandr libxi libxcursor libxinerama libxkbcommon \
    mesa glib2 glslang
```

#### Windows

Visual Studio 2022 or later with the "Desktop development with C++" workload, or
any MSVC toolchain plus Ninja. CMake and Ninja come with the VS installer.

### Build

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/Flyengine          # Linux
build\Flyengine.exe        # Windows
```

For Visual Studio, open the folder in the IDE and build — `CMakeSettings.json`
is configured for it.

### Build options

| Option | Default | Effect |
| --- | --- | --- |
| `FLYENGINE_ENABLE_FLYCLOUD` | `ON` | Builds the FlyCloud integration (vendored libcurl, miniz, nlohmann/json). Turning it off drops the libcurl build, which is the slowest part of a cold compile. |
| `FLYENGINE_ENABLE_CSHARP` | `ON` | C# scripting host. **Windows only** — see below. |
| `FLYENGINE_ENABLE_DESKTOP_VALIDATION` | `OFF` | Validate `packaging/*.desktop` and the MIME XML at build time. |
| `FLYENGINE_BUILD_TESTS` | `OFF` | Build the `tests/` targets. |
| `FLYENGINE_DATA_DIR` | `<prefix>/share/flyengine` | Where the binary looks for `assets/` and `shaders/`. Also the install destination, so the two cannot drift apart. |

---

## Running

```sh
Flyengine                 # splash screen, then the project manager
Flyengine /path/to/proj   # open a project folder
Flyengine /path/to/proj.flyproj
```

### Where the engine finds its data

The binary locates `assets/` and `shaders/` on its own, so it works from any
working directory — including a `.desktop` launcher, which starts with `$HOME`
as the CWD. The first of these that contains the file wins:

1. `$FLYENGINE_DATA_DIR`
2. the current directory, then up to four parent directories
3. the executable's own directory
4. `<exe>/../share/flyengine`, `<exe>/../share`, `<exe>/../lib/flyengine`,
   `<exe>/../../share/flyengine`
5. the `FLYENGINE_DATA_DIR` compiled in at configure time

To point a build at a different data tree without reinstalling:

```sh
FLYENGINE_DATA_DIR=/path/to/FlyEngine ./build/Flyengine
```

If nothing matches, the engine logs the exact list of roots it tried rather
than silently rendering nothing.

---

## Installing on Linux

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local
cmake --build build
sudo cmake --install build
```

Then refresh the desktop caches:

```sh
sudo update-desktop-database /usr/local/share/applications
sudo gtk-update-icon-cache -f -t /usr/local/share/icons/hicolor
sudo update-mime-database /usr/local/share/mime
```

This installs the binary, `assets/` and `shaders/` under
`/usr/local/share/flyengine`, a `.desktop` entry, the hicolor icon set, and a
MIME type so double-clicking a `.flyproj` file opens the editor.

### Relocatable tarball

`packaging/make_tarball.sh` builds a runtime-only tarball that can be unpacked
anywhere:

```sh
./packaging/make_tarball.sh            # build/ -> dist/
sudo tar -xzf dist/flyengine-linux-x86_64.tar.gz -C /
```

The unpacked binary runs in place (`./usr/local/bin/Flyengine`) without being
installed, because the data-root search is relative to the executable.

---

## Wayland

raylib 6 with the GLFW backend talks X11. Under a Wayland session this works
through **XWayland**, which GNOME, KDE Plasma and Sway enable by default. If
your session has XWayland disabled, enable it, or start the engine with
`GDK_BACKEND` / your compositor's X11 compatibility layer.

There is no native Wayland backend to select; this is a property of raylib's
windowing backend, not of Flyengine.

---

## Platform differences

Everything below is deliberate, not unfinished.

### File dialogs

The engine never calls a Win32 dialog. On Linux it shells out to
**`zenity`**, falling back to **`kdialog`**. If neither is installed, an ImGui
modal asks for the path.

Install one for native dialogs:

```sh
sudo apt install zenity        # or: sudo pacman -S zenity
```

Dialogs are **non-blocking**. A request is started with
`platform::Begin*Dialog()` and answered later by `platform::PollDialogResult()`
on the frame after it completes, so the render loop never stalls. Results are
tagged with a `DialogPurpose` so several subsystems can poll every frame
without consuming each other's results.

### C# scripting

**Windows only.** The CoreCLR host embeds `coreclr.dll` and is guarded by
`#if defined(_WIN32)`; on other platforms `LoadCoreCLR()` reports that
scripting is unavailable and the editor runs with it disabled.

This is a known gap, not a claim that it works. The managed assembly itself
builds on any platform:

```sh
./ScriptingSDK/build_sdk.sh [project_path]
```

The Linux CLR host is the next piece of work.

### Fonts

`arial.ttf` dropped in the working directory wins (so an in-tree build can be
tweaked), then the platform's candidates: Arial on Windows, DejaVu Sans and
Liberation Sans on Linux.

### The scripting ABI

The C# side P/Invokes `FlyNative_*` through the process's own exported symbols.
On Linux that means the executable must export its dynamic symbol table, which
CMake does with `ENABLE_EXPORTS ON`. Without it the build and link succeed and
every script call fails on the first tick — CI checks the export count
explicitly for this reason.

---

## Shader validation

GLSL errors are runtime-only by default: raylib's `LoadShader` logs a warning
and substitutes a fallback shader, so a broken shader can ship without any build
failing. `shaders/terrain.frag` did exactly that — it referenced two undeclared
uniforms and indexed a sampler array in a way GLSL 3.30 does not allow, and
terrain silently rendered with the fallback on every platform.

If `glslangValidator` is on `PATH`, the build now compiles every shader and
fails on error:

```sh
sudo apt install glslang-tools     # or: sudo pacman -S glslang
```

Without it the build prints a warning and continues. A GL context is never
created at build time, so this is static validation only — it does not prove a
shader links against a particular driver.

---

## Continuous integration

`.github/workflows/build.yml` runs on every push and pull request:

- **Linux** on `ubuntu:24.04` and `archlinux:base-devel` — the two distribution
  families the project targets. Configures, builds, checks the `FlyNative_*`
  export count, installs, and smoke-tests the installed binary from an unrelated
  working directory.
- **Wayland/XWayland** — starts the engine under `xvfb-run` and fails if GLFW
  cannot open a display, which is the failure mode when XWayland is missing.
- **Windows/MSVC** — builds, checks the export table with `dumpbin`, and fails
  on a dynamic CRT dependency (the build is configured for a static CRT so
  `VCRUNTIME140.dll` is not needed at runtime).
- **Tarball** — produces `flyengine-linux-x86_64.tar.gz` as a build artifact.

---

## Project layout

```
CMakeLists.txt            the single build definition for both platforms
include/Engine/Platform/  the cross-platform shim (see below)
src/Engine/Platform/      its implementation
shaders/                  GLSL, validated at build time
assets/                   editor icons, logo, preset and terrain textures
packaging/                .desktop entry, MIME XML, tarball script
ScriptingSDK/             the C# SDK and its build scripts
extern/box3d/             vendored Box3D
raylib/                   vendored raylib
```

### The platform shim

`include/Engine/Platform/Platform.hpp` is the only place the rest of the engine
should reach for anything platform-specific. It covers:

- **dialogs** — non-blocking `zenity`/`kdialog`/ImGui-prompt
- **processes** — `RunProcessCapture`, `LaunchDetached`
- **locations** — `ExecutablePath`, `UserHomeDir`, `ConfigDir`, `DocumentsDir`,
  `DataSearchRoots`, `ResolveAsset`, `ResolveShader`
- **desktop** — `RevealInFileManager`, `OpenWithDefaultApp`, font discovery
- **language** — `FLY_API` (export decoration) and `FLY_TRY`/`FLY_CATCH`
  (structured exception guards, which are real `__try`/`__except` on MSVC and
  degrade to a plain scope on GCC/Clang, which have no SEH)

`Window.h`-style headers are included *only* inside
`src/Engine/Platform/*.cpp` and behind `#if defined(_WIN32)`.

---

## Credits

* VisualCoder-sudo
* [raylib](https://www.raylib.com/) by Ramon Santamaría and contributors
* [Box3D](https://github.com/erincatto/box3d) by Erin Catto
* [Dear ImGui](https://github.com/ocornut/imgui) by Omar Cornut
* [ufbx](https://github.com/ufbx/ufbx) by Ulti
