# Oficial Repo of Slim Executor Releases

**Slim Executor** is a Windows desktop workspace for writing and running Lua scripts locally, backed by a reusable C++17 Lua runtime. The project also provides a CMake SDK so other C++ applications can embed the same runtime.

> **Scope:** Slim Executor runs Lua in its own application/runtime. It does not inject into Roblox, attach to third-party processes, or bypass another application's protections. Do not run scripts you do not trust.

## What is it for?

Use Slim Executor to experiment with Lua, learn the language, test small scripts, and integrate Lua scripting into your own C++ software.

## Features

### Desktop application

- Native Windows desktop application with a dark Qt 6 interface (Qt build).
- Tabbed Lua editor with basic syntax highlighting, line numbers, and a live line/column indicator.
- Interactive Lua scripts can ask for text with `prompt(message)`; cancelling returns `nil`.
- Qt desktop scripts can create native windows with labels, text fields, password and multiline inputs, buttons, checkboxes, sliders, dropdowns, color pickers, drawing canvases, and timer-driven Lua callbacks through `ui.window(...)`.
- Canvas callbacks can read keyboard presses/releases and mouse movement, clicks, and wheel input. Keyboard names include arrows, WASD, common editing keys, digits, letters, F1–F12, and modifier combinations such as `Ctrl+Shift+S`.
- Lua windows can poll connected XInput controllers with `window:gamepad(index)`, returning stick axes, triggers, and button states; controller indices run from 0 to 3.
- UI windows can pause/resume timers, persist small text values between app launches, open/save files, show messages, play audio, and export the Scripts workspace as a ZIP archive.
- Run scripts with **F5** or **Ctrl+Enter**; open and save with **Ctrl+O** and **Ctrl+S**.
- Open and save `.lua` files.
- Right-side workspace Explorer for Lua/text files, with nested folder creation and deletion plus file rename.
- Integrated console below the editor for script output and errors, with elapsed time reported for each run.
- Integrated Windows command terminal with standard output/error display, command input, clear action, and `Ctrl+Shift+T` shortcut.
- Experimental OpenGL 3D preview with a lit rotating cube, floor grid, orbit camera (left mouse drag), and zoom (mouse wheel).
- Copy the latest Lua error, a selected console section, or all console output from the console controls/context menu.
- Reset the Lua runtime or clear the output panel.
- Run, reset, and clear controls beside **Help**; file actions remain in **File** and use standard keyboard shortcuts.
- Checks GitHub Releases for newer stable versions at startup and from **Help → Check for updates**. **Help → Download available versions** lists downloadable installers in the current version line. The app downloads the selected official setup, verifies its SHA-256 digest published by GitHub, and asks before launching the installer.

### C++ runtime SDK

- Reusable C++17 static library built around Lua and sol2.
- Persistent Lua state between executions, until reset.
- Captured output through standard Lua `print(...)` and the host helper `log(message)`.
- Standard Lua `coroutine` and `utf8` libraries, with `require(...)` for standard libraries and installed local Lua modules.
- Local library manager under **Help → Manage Lua libraries** to search installed modules, import `.lua` files, replace modules, and remove them.
- Host helper `add(a, b)`.
- Execution results containing success status, output, error text, and elapsed execution time in milliseconds (`duration_ms`).
- CMake install/export package for external C++ projects.
- Automated runtime tests and external-consumer checks in GitHub Actions.

## Current status

The Qt 6 desktop interface uses the Kali Red Dark palette, square corners, a tabbed editor with a console below it, and a full-height script Explorer on the right. Check [GitHub Actions](https://github.com/K3NYEL/Slim_Executor/actions) for CI status and [Releases](https://github.com/K3NYEL/Slim_Executor/releases) for the official end-user package.

## Install on Windows

For an official release, open [GitHub Releases](https://github.com/K3NYEL/Slim_Executor/releases) and download the only published asset: **`SlimExecutor-Setup-vX.Y.Z-windows-x64.exe`**. Run the installer and follow the steps. It installs for your Windows user account, creates a Start Menu shortcut, and offers an optional desktop shortcut. Administrator permission is not required.

To test an untagged branch build, open [GitHub Actions](https://github.com/K3NYEL/Slim_Executor/actions), choose a successful workflow run, and download its artifact.

## Build the Qt 6 desktop app

### Requirements

- Windows 10 or 11, 64-bit
- Git
- Visual Studio 2022 or Build Tools with **Desktop development with C++**
- CMake 3.21 or newer
- vcpkg, with `VCPKG_ROOT` set to its installation directory
- Qt 6 for MSVC 2022 64-bit

Clone the repository and open PowerShell in its directory:

```powershell
git clone https://github.com/K3NYEL/Slim_Executor.git
cd Slim_Executor
```

Configure and build the Qt application:

```powershell
cmake -S . -B build-qt `
  -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
  -DVCPKG_TARGET_TRIPLET=x64-windows `
  -DSLIM_EXECUTOR_BUILD_QT_DESKTOP=ON `
  -DBUILD_TESTING=OFF

cmake --build build-qt --config Release --parallel
```

This configuration expects Qt 6 Widgets and Network to be available to CMake. To build the SDK and native fallback instead, omit `-DSLIM_EXECUTOR_BUILD_QT_DESKTOP=ON` and use the static vcpkg triplet configured in the workflow.

## Use the Lua runtime in your C++ project

After building and installing the SDK, link the exported CMake target:

```cmake
find_package(SlimExecutor CONFIG REQUIRED)
target_link_libraries(MyApplication PRIVATE SlimExecutor::SlimExecutor)
```

Example usage:

```cpp
#include <slim_executor/lua_runtime.hpp>
#include <iostream>

int main() {
    slim::LuaRuntime runtime;
    const auto result = runtime.execute("log(\"Hello from Lua\"); return 2 + 3");

    if (!result.output.empty()) {
        std::cout << result.output << '\\n';
    }
    if (!result.error.empty()) {
        std::cerr << result.error << '\\n';
    }
    return result.success ? 0 : 1;
}
```

A single `LuaRuntime` instance keeps its Lua state between calls. Call `reset()` to recreate that state. The runtime enables Lua’s `base`, `math`, `string`, `table`, `coroutine`, and `utf8` libraries plus `package`/`require`. For example, `local coroutine = require("coroutine")` imports the coroutine library. In the desktop app, use **Help → Manage Lua libraries** to import local `.lua` modules; a file named `text_utils.lua` can be loaded with `require("text_utils")`. Imported modules are local files and are not independently security-reviewed. The runtime is not a security sandbox; do not run untrusted scripts. The `io` and `os` libraries are not opened by default.

## Runtime API

| API | Purpose |
| --- | --- |
| `LuaRuntime::execute(script)` | Execute a Lua script and return its result. |
| `LuaRuntime::reset()` | Reset the Lua state. |
| `ExecutionResult::success` | Indicates whether execution succeeded. |
| `ExecutionResult::output` | Captured log output and returned value, when available. |
| `ExecutionResult::error` | Compilation or runtime error information. |

The host-provided `log(message)` function writes to the result output. The host-provided `add(a, b)` helper returns the sum of two numbers.

In the Qt desktop app, `prompt(message)` opens a text input dialog and returns the entered text. If the user cancels, it returns `nil`. Other applications embedding the runtime can provide their own input handler with `LuaRuntime::set_input_handler`.

Try the interactive example at [`examples/interactive_cave.lua`](examples/interactive_cave.lua).

Qt desktop scripts can also create a small native interface:

```lua
local window = ui.window("My tool")
local input = window:input("Value", "10")
local result = window:label("Result: waiting")

window:button("Double", function()
    local value = tonumber(input:value()) or 0
    result:set_text("Result: " .. (value * 2))
end)

window:show()
```

See [`examples/gui_controls_showcase.lua`](examples/gui_controls_showcase.lua) for a complete interactive demo of the added controls, canvas input, storage, audio, dialogs, gamepad polling, and export.

For an animated canvas, `window:timer(interval_ms, callback)` calls the callback with elapsed seconds until stopped or an error occurs. Use `timer:pause()` and `timer:resume()` to temporarily suspend and continue it. A canvas supports drawing methods plus `on_key(callback)` and `on_mouse(callback)`. Mouse callbacks receive `(event, x, y, button, wheel)`; events are `move`, `press`, `release`, or `wheel`, with button IDs 1 (left), 2 (right), and 3 (middle).

Windows also provide `window:checkbox`, `slider`, `select`, `textarea`, `password`, and `color_picker` controls. Each returns a control with `value()` and, where applicable, `set_value(text)`. `window:save(key, value)` and `window:load(key)` persist text values in the app's settings. File dialogs use `window:open_file(filter)` and `window:save_file(filter)`; `window:message(title, text)` displays a message. Audio methods are `window:play_sound(path)`, `window:play_music(path, loop)`, and `window:stop_music()`. `window:export_project(zip_path)` packages the app's Scripts workspace.

`window:gamepad(index)` returns `nil` if the controller is disconnected; otherwise it returns an array with left/right stick axes, left/right triggers, A/B/X/Y, shoulder buttons, D-pad directions, Start, and Back in that order. Poll it from a timer callback for continuous movement.

The `ui` API is available in the Qt desktop app. Other programs embedding the runtime must provide their own UI host. See [`examples/gui_calculator.lua`](examples/gui_calculator.lua) for a complete widget example.

## Run tests

For the SDK/native build on Windows, configure the project with testing enabled, build it, then run:

```powershell
ctest --test-dir build -C Release --output-on-failure
```

GitHub Actions also checks the runtime tests and verifies that an external CMake consumer can use the installed SDK.


## Versioning and releases

Slim Executor uses Semantic Versioning. The single source of truth is `version-semver` in `vcpkg.json`; CMake generates the public version header and Windows file metadata from it.

- C++ API: include `<slim_executor/version.hpp>` and call `slim::version_string()` or `slim::version()`.
- Console demo: run `SlimExecutorDemo --version`.
- Qt desktop app: the window title follows the active script; the version is generated from the manifest.
- Version helper: `cmake -P scripts/version.cmake` prints the current version.

Add changes under `## [Unreleased]` in `CHANGELOG.md`, then preview a patch bump with:

```powershell
cmake -DACTION=bump -DPART=patch -DDRY_RUN=ON -P scripts/version.cmake
```

When ready to publish, remove `-DDRY_RUN=ON`, then push the commit and tag with `git push --follow-tags`. The Release workflow validates the tag against `vcpkg.json`, checks the changelog, builds/tests the Qt desktop app, and publishes only the Windows setup installer (`SlimExecutor-Setup-vX.Y.Z-windows-x64.exe`). Older stable releases are cleaned of other downloadable assets when the release workflow next runs. Do not reuse or move an existing release tag.

## Version reset note

The development version was reset to `0.1.0` for a new `0.1.x` line. The historical `v0.4.0` release remains available, and the official `v0.1.0` release is published separately. Do not move or reuse existing tags. The update checker compares releases within the same `0.x` minor line so that releases from an incompatible line are not incorrectly offered as updates.

## Security notes

The runtime is intended for scripts you trust. Restricting the Lua libraries is not a complete security sandbox. Executing hostile or untrusted scripts in-process can expose the host application to risk. For untrusted code, use a separate low-privilege process with operating-system-enforced resource, filesystem, and network restrictions.

## Project structure

```text
include/slim_executor/   Public C++ API
src/lua_runtime.cpp      Lua runtime implementation
src/main.cpp             Console demo host
src/qt_main.cpp          Qt 6 desktop interface
src/gui_main.cpp         Native Windows fallback interface
examples/interactive_cave.lua  Example using prompt() for player choices
examples/gui_calculator.lua    Example creating a Qt window from Lua
examples/gui_bouncing_ball.lua Animated canvas example with gravity and collisions
examples/gui_orbital_defense.lua Keyboard-controlled arcade game example
tests/                   Runtime and SDK consumer tests
.github/workflows/       Automated build and test workflow
```
