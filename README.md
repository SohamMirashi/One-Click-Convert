# RadialConvert (working title)

A local, offline file-conversion tool for Windows. Hold **Win + Alt**, **left-click** any file, and the screen dims, the file's enlarged icon flies to the center, and a radial wheel of valid conversion targets appears around it. Pick a slice and a **new converted copy** is saved next to the original. The original is never modified.

> Status: design/planning stage. This README records every decision made so far and the plan for building it.

---

## Table of contents

1. [Goals and non-goals](#1-goals-and-non-goals)
2. [Feasibility summary](#2-feasibility-summary)
3. [Core design decisions](#3-core-design-decisions)
4. [User experience](#4-user-experience)
5. [Technology stack](#5-technology-stack)
6. [Qt Quick first, Direct2D/DirectComposition only if needed](#6-qt-quick-first-direct2ddirectcomposition-only-if-needed)
7. [Architecture and project structure](#7-architecture-and-project-structure)
8. [Conversion engine](#8-conversion-engine)
9. [Windows integration details](#9-windows-integration-details)
10. [Runtime flow](#10-runtime-flow)
11. [Build flow (in order)](#11-build-flow-in-order)
12. [Feature list](#12-feature-list)
13. [Things deliberately left out](#13-things-deliberately-left-out)
14. [Known risks and gotchas](#14-known-risks-and-gotchas)
15. [Licensing notes](#15-licensing-notes)
16. [Future work](#16-future-work)
17. [Decision log](#17-decision-log)

---

## 1. Goals and non-goals

### Goals

- Convert a file to another format in one fast gesture, without opening any app or website.
- Work **entirely offline** and locally.
- Save the result as a **new copy**; never overwrite or alter the original.
- Offer **only conversions that are actually supported** for the clicked file's type.
- Feel polished: dimmed screen, smooth icon animation, a wheel that adapts to the file type.

### Non-goals (for now)

- Not a commercial product. This is a personal project at this stage.
- Not cross-platform in the first version (Windows only; other OSes come later).
- Not a file editor (no video editing, document editing, or image editing).
- Not a cloud or AI-powered converter.

---

## 2. Feasibility summary

The idea is **feasible**, and **C++ is a good fit**.

| Area | Verdict | Notes |
|---|---|---|
| Global gesture (Win + Alt + left click) | Feasible | Low-level keyboard and mouse hooks. |
| Finding the clicked file | Feasible | Explorer COM interfaces, optionally UI Automation. |
| Dimmed screen and animated wheel | Feasible | Qt Quick (QML) handles this well. |
| Conversion engine | Feasible, with limits | Orchestrate existing libraries and tools rather than writing converters. |
| "Works for all file types" | **Not feasible as stated** | Rescoped to "every type that has a registered, valid conversion". |

---

## 3. Core design decisions

1. **Supported conversions only.** A conversion is offered only if it is valid for the detected file type. Examples: JPEG to MP3 is never shown, MP4 to PDF is never shown. The wheel is generated from a compatibility table, never hardcoded.
2. **Windows first.** Linux (Lubuntu) and other systems are planned later. Platform-specific code is isolated behind interfaces so a port does not touch the core.
3. **Trigger: Win + Alt + left mouse click.** The hotkey is fixed for now. A user-selectable hotkey comes later.
4. **Output is always a new copy**, named safely (`name (1).ext` if the target exists).
5. **Local and offline.** All conversion tools run on the user's machine.
6. **C++ for logic and backend.** The UI layer is Qt 6 with QML (Qt Quick).
7. **Qt Quick is the default for all visuals.** Direct2D/DirectComposition is used only where Qt Quick cannot do the job (see section 6).
8. **File type detection is by content, not extension**, so a PNG renamed to `.jpg` is still treated as a PNG.

---

## 4. User experience

1. The user holds **Win + Alt** and **left-clicks** a file in Explorer or on the Desktop.
2. The screen **dims**, similar to the Windows screenshot tool.
3. The file's **enlarged icon (or real thumbnail for images) and name** animate smoothly from where the file was to the **center of the screen**.
4. A **wheel expands around it**. Its slices depend on the file type:
   - PNG: JPG, WebP, BMP, PDF, ZIP, ...
   - MP4: MKV, GIF, MP3 (extract audio), ZIP, ...
   - DOCX: PDF, ZIP, ...
5. The user clicks a slice (or releases the button over one). A progress notification appears.
6. A new file is saved beside the original. The overlay closes.
7. **Esc** or a click outside the wheel cancels with no changes.

Other behavior:

- Types with many targets use a **nested sub-wheel** (about 6 to 8 slices is the comfortable limit per ring).
- Selecting multiple files shows only targets valid for **all** of them.
- Lossy conversions (for example PNG with transparency to JPEG) show a brief warning.

---

## 5. Technology stack

| Concern | Choice |
|---|---|
| Language | C++20 |
| Build system | CMake |
| Package management | vcpkg |
| Compiler | MSVC (Visual Studio) |
| UI | Qt 6 + QML (Qt Quick) |
| Native escape hatch | Direct2D / DirectComposition (only if needed) |
| Windows integration | Win32 low-level hooks, Shell COM, UI Automation |
| Images | libvips or Magick++ (ImageMagick) |
| Audio and video | FFmpeg |
| Documents | Pandoc, LibreOffice headless |
| PDF | Poppler or MuPDF |
| Archives and compression | libarchive, zlib, zstd, xz |
| Type detection | libmagic (or Qt's `QMimeDatabase`) |
| Tests | GoogleTest or Catch2 |
| Packaging | `windeployqt` plus an installer |

---

## 6. Qt Quick first, Direct2D/DirectComposition only if needed

**Rule:** build every visual in Qt Quick first. Reach for Direct2D/DirectComposition only when Qt Quick cannot do something, or does it badly enough to hurt the experience.

What Qt Quick is expected to handle on its own:

- The transparent, frameless, always-on-top, full-screen overlay window
- The screen dimming (a dark semi-transparent rectangle, or a captured screenshot drawn dimmed)
- Blur effects on a captured screenshot (QML effects/shaders)
- The icon fly-to-center animation, easing, and staggered wheel expansion
- Hover, selection, and sub-wheel behavior

Qt's `QScreen::grabWindow` can capture the screen for the dim/blur backdrop.

**Candidate cases for the native layer** (decide only after prototyping, with evidence):

- Live blur of the real desktop behind the overlay, if the captured-screenshot approach looks wrong or lags
- Frame-pacing or tearing problems on the overlay across multiple monitors with mixed DPI
- Capture limitations (protected content, exclusive fullscreen) that Qt's grab cannot handle
- Any effect where Qt Quick measurably stutters

**Not a reason to use Direct2D:** the global hooks, Explorer selection, UI Automation, and shell thumbnails. Those are Win32/COM work and live in the platform module regardless.

If a native layer is added, it goes in a clearly separated optional module (see structure below), and the QML overlay sits on top of it as a transparent window.

---

## 7. Architecture and project structure

Dependencies only point downward. `core` knows nothing about Windows or Qt. `platform` implements interfaces. `ui` talks to `core` only through the bridge. `app` wires everything together.

```
RadialConvert/
├─ CMakeLists.txt
├─ vcpkg.json
│
├─ core/                          (pure C++20: no Qt, no Windows headers)
│  ├─ registry/
│  │   ├─ Converter.h                 (inputs, outputs, run())
│  │   ├─ ConverterRegistry.{h,cpp}   (targetsFor(type))
│  │   └─ FormatCatalog.{h,cpp}       (extension -> MIME -> family)
│  ├─ detect/
│  │   └─ TypeDetector.{h,cpp}        (content-based detection)
│  ├─ jobs/
│  │   ├─ Job.h
│  │   ├─ JobQueue.{h,cpp}            (thread pool, progress, cancel)
│  │   └─ OutputPathResolver.{h,cpp}  (name (1).ext, never overwrite)
│  └─ converters/
│      ├─ image/      (libvips / Magick++)
│      ├─ media/      (FFmpeg)
│      ├─ document/   (Pandoc / LibreOffice headless)
│      ├─ pdf/        (Poppler / MuPDF)
│      └─ archive/    (libarchive)
│
├─ platform/
│  ├─ IPlatform.h                 (ISelectionProvider, IInputTrigger,
│  │                               IThumbnailProvider, IScreenCapture)
│  └─ windows/
│      ├─ input/
│      │   ├─ HookThread.{h,cpp}        (WH_KEYBOARD_LL + WH_MOUSE_LL)
│      │   ├─ ModifierState.h
│      │   └─ MaskKeyInjector.cpp       (stops Start menu / Alt menu bar)
│      ├─ shell/
│      │   ├─ ExplorerSelection.{h,cpp} (IShellWindows, IFolderView2)
│      │   ├─ ShellThumbnail.{h,cpp}    (IShellItemImageFactory)
│      │   └─ ItemBounds.{h,cpp}        (UI Automation bounding rect)
│      ├─ display/
│      │   └─ MonitorLayout.{h,cpp}     (virtual screen, per-monitor DPI)
│      └─ native_gfx/                   (OPTIONAL: Direct2D/DirectComposition,
│                                        only added if Qt Quick proves insufficient)
│
├─ ui/
│  ├─ bridge/                     (the only place C++ meets QML)
│  │   ├─ OverlayController.{h,cpp}     (QObject: state machine)
│  │   ├─ WheelModel.{h,cpp}            (QAbstractListModel of slices)
│  │   ├─ ScreenGrabber.{h,cpp}         (captures the screen for the backdrop)
│  │   └─ ThumbnailImageProvider.{h,cpp}
│  └─ qml/
│      ├─ Main.qml                  (transparent overlay window)
│      ├─ Backdrop.qml              (dim and blur)
│      ├─ CenterIcon.qml            (fly-to-center animation)
│      ├─ Wheel.qml
│      ├─ Slice.qml
│      ├─ SubWheel.qml
│      └─ Theme.qml                 (colors, durations, easing curves)
│
├─ app/
│  ├─ main.cpp                    (startup, tool detection, wiring)
│  ├─ AppController.{h,cpp}       (trigger -> selection -> overlay -> job)
│  ├─ TrayIcon.{h,cpp}
│  └─ Settings.{h,cpp}            (JSON config: hotkey, theme, output folder)
│
├─ resources/                     (icons per target format, qrc)
└─ tests/                         (core unit tests, converter tests with sample files)
```

### Why this shape

- `core` stays testable without any GUI or OS dependency.
- Porting to Linux later means adding `platform/linux/` with the same interfaces; `core` and the QML do not change.
- All visual tuning lives in `Theme.qml`, so the look and feel can change without recompiling the C++.

---

## 8. Conversion engine

### Type families

| Family | Example formats |
|---|---|
| image | png, jpg, webp, bmp, tiff |
| audio | mp3, wav, flac |
| video | mp4, mkv, gif |
| document | docx, odt, md, html |
| pdf | pdf |
| archive | zip, tar, 7z |

### Converter registry

Each converter declares what it accepts and produces:

```cpp
struct Converter {
    std::string id;                       // e.g. "image_magick"
    std::set<std::string> inputs;         // {"png","jpg","webp","bmp"}
    std::set<std::string> outputs;        // {"jpg","png","webp","pdf"}
    std::function<bool(const Job&)> run;  // does the conversion
};
```

The wheel is built from the registry:

```cpp
std::vector<std::string> targetsFor(const std::string& inputFmt) {
    std::set<std::string> result;
    for (auto& c : registry)
        if (c.inputs.count(inputFmt))
            result.insert(c.outputs.begin(), c.outputs.end());
    result.erase(inputFmt);   // never offer png -> png
    return {result.begin(), result.end()};
}
```

Adding a new format means adding a registry entry, not touching the UI.

### Cross-family conversions that make sense

These are explicit registry entries:

- video to audio (extract), video to GIF
- image to PDF, PDF to images
- any file or folder to ZIP / 7z (the "compressed file" option)

### Rules

- **Tool detection at startup:** the app checks for FFmpeg, LibreOffice, Pandoc, and so on. If a tool is missing, its converters are dropped so those slices never appear.
- **Never overwrite:** the output name is resolved safely.
- **Run on worker threads:** the UI never blocks.
- **Clean up on failure:** partial output files are deleted.
- **Chained conversions** (docx to pdf to png) are out of scope for v1. If added later, model formats as a graph and find the shortest path.
- **Unknown file types:** show only universal options (ZIP, 7z), or nothing.

---

## 9. Windows integration details

### Gesture detection

- `RegisterHotKey` cannot be used, because it needs a non-modifier key and this gesture is modifiers plus a mouse click.
- Run two low-level hooks on a **dedicated thread with its own message loop**:
  - `WH_KEYBOARD_LL` to track Win and Alt state
  - `WH_MOUSE_LL` to catch the left-button press
- Keep hook callbacks tiny: set a flag and post a message to the main thread. Slow hooks get removed by Windows.
- **Start menu and Alt menu-bar suppression:** releasing Win alone opens the Start menu, and releasing Alt alone focuses the menu bar. Inject a harmless unassigned virtual-key press (as AutoHotkey does) so Windows treats the keys as used in a combo.

### Finding the clicked file

- **Approach A (v1):** let the click pass through so Explorer selects the item, wait a few milliseconds, then read the selection through COM (`IShellWindows`, `IFolderView2::GetSelection`). This returns full paths, handles hidden file extensions, supports multi-select, and works on the Desktop.
- **Approach B (possible refinement):** swallow the click and use UI Automation `ElementFromPoint`. This avoids accidental drags but requires mapping a display name back to a real path.

### Icon and animation inputs

- **Large thumbnail:** `IShellItemImageFactory::GetImage` (up to 256 px). Images return a real thumbnail, not just a generic icon.
- **Start position of the animation:** the item's on-screen rectangle (UI Automation bounding rect), falling back to the cursor position.

### Display

- Cover the entire **virtual screen** (all monitors).
- Handle **per-monitor DPI** from the start; it is painful to retrofit.

---

## 10. Runtime flow

1. The hook thread sees Win + Alt held plus a left click and posts a message to the main thread.
2. `ExplorerSelection` returns the selected paths; `ItemBounds` returns the icon rectangle.
3. `TypeDetector` identifies the file; the registry returns the valid targets.
4. The overlay appears: the screen is captured and dimmed (blur optional).
5. The icon flies from its original position to the screen center; the file name fades in.
6. The wheel expands around the icon with a staggered animation.
7. The user selects a slice; `JobQueue` runs the converter on a worker thread.
8. The overlay closes; a notification reports success or failure.
9. The new copy sits beside the original.

### Overlay state machine

```
Idle -> Entering -> WheelOpen -> (SubWheel) -> Converting -> Closing -> Idle
                        |
                        +-- Esc / click outside --> Closing -> Idle
```

### Threading rules

- Hooks run on their own thread.
- Conversions run on worker threads.
- All updates to QML go through Qt's **queued signals**, never directly from worker threads.

---

## 11. Build flow (in order)

### Phase 1: Foundation

1. Set up the toolchain: Visual Studio (MSVC), CMake, vcpkg, and Qt 6 (MSVC kit). Get an empty project building with all targets linking.
2. Add dependencies through `vcpkg.json`: libvips (or Magick++), FFmpeg, libarchive, libmagic, and a test framework.

### Phase 2: Core logic (no UI)

3. Write `FormatCatalog` and `TypeDetector`, with tests using real sample files, including ones with wrong extensions.
4. Write `Converter`, `ConverterRegistry`, and `targetsFor(type)`.
5. Implement the first converter (image: PNG, JPG, WebP, BMP, PDF) and drive it from a small command-line executable.
6. Add `JobQueue` and `OutputPathResolver`.
7. Add the remaining converter families one at a time (media, document, pdf, archive), each registered only if its tool is found.

### Phase 3: Windows integration

8. Build `HookThread` and `ModifierState` to detect Win + Alt + left click, including the mask-key injection.
9. Build `ExplorerSelection` and verify it on Explorer windows and on the Desktop.
10. Build `ShellThumbnail` and `ItemBounds`.
11. Build `MonitorLayout` with correct per-monitor DPI handling.

### Phase 4: Qt Quick overlay

12. Create the transparent, frameless, topmost QML window covering the virtual screen.
13. Implement `ScreenGrabber` and `Backdrop.qml` (dim, optional blur).
14. Write `OverlayController` as the state machine above.
15. Build `CenterIcon.qml` (fly-to-center) and the file name fade-in.
16. Build `WheelModel`, `Wheel.qml`, and `Slice.qml` with staggered expansion and hover.
17. Add `SubWheel.qml` for types with many targets.
18. **Evaluate the result.** If something is impossible or insufficient in Qt Quick (see section 6), add the optional `native_gfx` module for that specific gap only.

### Phase 5: Connect everything

19. In `AppController`, wire the full chain: trigger, selection, type detection, registry targets, overlay, slice choice, job, notification.
20. Handle every exit: Esc, click outside, and release of the modifier keys.
21. Add multi-file selection (targets valid for all selected files).

### Phase 6: Hardening and packaging

22. Error handling: delete partial output on failure, show clear messages, warn on lossy conversions.
23. Tray icon, settings file, user-selectable hotkey, and a "run at startup" option.
24. Test edge cases: elevated windows, multiple monitors with different DPI, network drives, files with no extension, very large files.
25. Package with `windeployqt`, bundle the converter tools, and build an installer.

### De-risk first

The two riskiest areas are the **hooks plus Explorer selection** (steps 8 to 10) and the **overlay's focus and transparency behavior** (steps 12 to 13). Prove both with throwaway prototypes before building everything else around them.

---

## 12. Feature list

### Included

- Win + Alt + left click gesture
- Dim screen, flying icon, and a dynamic wheel per file type
- Real thumbnails for image files
- Content-based type detection
- Supported-only conversions
- Safe output naming; originals never touched
- Background conversion with progress notification
- Multi-file selection
- Sub-wheels for types with many targets
- Tray icon and settings file

### Planned additions

- User-selectable hotkey
- Quality and size presets (for example "compress to under 1 MB", "resize to 50%")
- Undo (deletes the copy that was just created)
- User-defined conversions through config command templates (for example `ffmpeg -i {in} {out}`)
- Lossy-conversion warnings
- Option to strip EXIF metadata
- Folder to ZIP
- Offline OCR (Tesseract)
- Favorites and ordering of wheel slices
- Configurable output folder

---

## 13. Things deliberately left out

- "Works for every file type" as a promise
- Replacing Explorer's native context menu (fragile, especially on Windows 11)
- Cloud or AI-based conversion
- Editing features of any kind
- Chained multi-step conversions in v1
- Linux and other operating systems in v1

---

## 14. Known risks and gotchas

- **Win + Alt is a crowded combination.** Windows and apps use many Win-key shortcuts (Game Bar uses Win + Alt + other keys). Keep the hotkey configurable in the future.
- **Antivirus false positives.** Global hooks resemble keyloggers. Expect to add a test exclusion; code-sign if distributed.
- **Elevated windows.** A non-admin hook cannot interact with apps running as administrator (UIPI) unless the tool is also elevated.
- **Overlay focus.** Windows' foreground-lock rules can stop a topmost window from receiving keyboard focus. Attaching to the foreground thread's input and calling `SetForegroundWindow` usually fixes it.
- **Per-monitor DPI.** Mixed-DPI setups need handling from the start.
- **Lossy conversions.** Some conversions lose data (transparency, quality). Warn the user.
- **Long conversions.** Must run asynchronously with cancel support.
- **Hidden file extensions.** Another reason to prefer reading the Explorer selection over matching display names.
- **Tool availability.** Converters depend on external tools; detect them at startup and hide unsupported slices.

---

## 15. Licensing notes

This is a personal project for now, but if it is ever distributed:

- **Qt** is available under LGPL (dynamic linking) or commercial terms.
- **FFmpeg** is LGPL or GPL depending on how it is built.
- **LibreOffice** (MPL), **Pandoc** (GPL), and other bundled tools each have their own terms.
- Review each dependency's license before any public release.

---

## 16. Future work

- Linux support, starting with Lubuntu (LXQt, PCManFM-Qt). The file manager's custom actions can pass selected files to the tool, and global shortcuts work on X11 sessions. Wayland restricts global hotkeys and popups at the cursor.
- A settings UI.
- Chained conversions through a format graph.
- Plugin system for third-party converters.

---

## 17. Decision log

| # | Decision |
|---|---|
| 1 | The tool is feasible and will be written in C++. |
| 2 | "All file types" is replaced by "only supported conversions"; unsupported targets never appear. |
| 3 | The conversion engine orchestrates existing offline libraries and tools. |
| 4 | Personal, non-commercial project for now. |
| 5 | Windows first; other operating systems later. |
| 6 | Trigger is Win + Alt + **left** click (changed from right click). |
| 7 | Hotkey is fixed for now; user-selectable hotkey later. |
| 8 | UI: dim the screen like the screenshot tool, animate the enlarged icon and name to the center, and show a dynamic wheel that differs per file type. |
| 9 | UI stack is **Qt 6 + QML (Qt Quick)**. Direct2D/DirectComposition is used only where Qt Quick fails or is insufficient. |
| 10 | Backend and logic are in C++, with a pure `core` library independent of Qt and Windows. |
| 11 | Project follows the structure in section 7 and the build flow in section 11, with no fixed timelines. |
