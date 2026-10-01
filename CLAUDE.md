# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

wfview is a Qt (5 or 6) C++17 desktop application for controlling Icom, Kenwood and Yaesu amateur radios (CAT/CI-V control, spectrum/waterfall, network audio) on Linux, macOS and Windows. `wfserver` is a headless build of the same sources that serves a radio over the network (Icom only today).

## Build

The build uses qmake. Do an out-of-tree build from a sibling directory:

```bash
mkdir build && cd build
qmake ../wfview/wfview.pro            # add CONFIG+=release / CONFIG+=debug, PREFIX=/opt, VERSION="x.y"
make -j
sudo make install                     # installs binary, rigs/, qdarkstyle/, .desktop, icons under $PREFIX
```

- Use `qmake6` for Qt6 and `qmake`/`qmake-qt5` for Qt5. Linux dependencies are listed in `INSTALL.md` and `WFSERVER.md`. `tools/fullbuild-wfview.sh` is a full Debian build script.
- Build the headless server with `qmake ../wfview/wfserver.pro`. It defines `BUILD_WFSERVER` (wfview.pro defines `BUILD_WFVIEW`) and `FTDI_SUPPORT`. Shared sources use `#ifdef` on these macros, so changes to shared code must compile under both project files.
- **Every new source file must be added by hand to `SOURCES`/`HEADERS`/`FORMS` in `wfview.pro`**, and also in `wfserver.pro` if the server uses it.
- On Windows and macOS, the build expects third-party dependencies checked out as **sibling directories** of the repo: `../qcustomplot`, `../portaudio`, `../rtaudio`, `../opus`, `../eigen`, `../hidapi` and `../r8brain-free-src`. On Linux they come from system packages. qcustomplot's library name differs by distro and Qt version, so the .pro file probes `ldconfig` for it.
- CI is `.github/workflows/build-and-release.yml`. It builds the Windows (qmake + MSVC; dependencies built with cmake), macOS and Linux AppImage packages. The `*.bak` workflows are disabled.
- The repo has no automated test suite.

## Architecture

**Threading model.** `wfmain` (`src/wfmain.cpp`, a very large file) is the main window and does most of the wiring. It creates the radio backend, the servers, the USB controller, the cluster client and TCI, and moves each one to its own `QThread`. The objects talk to each other only through Qt signals/slots.

**Radio backends (`src/radio/`, headers in `include/`).** `rigCommander` is an abstract base class. It declares the rig-agnostic signals and virtual slots. `wfmain` creates one of `icomCommander`, `kenwoodCommander` or `yaesuCommander` based on the manufacturer:
- Icom: CI-V over serial, or over LAN using the `icomUdp*` classes (`icomUdpHandler` for control, plus separate CI-V and audio streams).
- Yaesu: CAT, plus the `yaesuUdp*` classes for the LAN protocol (control/cat/audio/scope).
- `*Server` classes (subclasses of `rigServer`) emulate the radio's network protocol, so other wfview instances or clients can connect to this one.

**Command abstraction: `funcs` + `cachingQueue`.** This is the central concept.
- `include/wfviewtypes.h` defines `enum funcs` (every radio function: `funcFreqSet`, `funcModeGet`, …) and a parallel `funcString[]` array of display names. **The two must stay in the same order.** Add new entries before `funcLastFunc`.
- `cachingQueue` (`include/cachingqueue.h`) is a singleton `QThread` (`cachingQueue::getInstance()`). UI widgets and servers call `queue->add(priority, funcX, recurring, receiver)` or `addUnique(...)`. They never call the rig directly. The queue emits `haveCommand` to the active `rigCommander`. That backend encodes the command using the rig's capability table and sends it. It decodes replies back into `funcs` values and passes them to `queue->receiveValue(...)`, which updates the cache and emits `cacheUpdated(cacheItem)`.
- Consumers (widgets, `rigctld`, `tciServer`, `bandbuttons`, …) connect to `cacheUpdated` and `rigCapsUpdated(rigCapabilities*)`. Priorities use prime numbers so that the recurring polls at each level interleave.

**Rig definition files (`rigs/*.rig`).** Each supported radio is described by a QSettings INI file, not by code. The file contains model info, CI-V address, capabilities, a `Commands` array and a `Periodic` array (recurring polls with a priority). Each backend's rig-caps loader (for example `icomCommander`, around `beginReadArray("Commands")`) matches each command's `Type=` string against `funcString[]`, case-insensitively, to fill `rigCapabilities` (`include/rigidentities.h`). Consequences:
- Adding support for a radio, or a command for an existing radio, is usually a `.rig` edit. The in-app Rig Creator (`src/rigcreator.cpp`) edits these files.
- Renaming an entry in `funcString[]` breaks every `.rig` file that references it. An unmatched name logs "Function … Not Found, rig file may be out of date?".
- Rig files are loaded from `<app>/rigs` (on macOS, `Contents/Resources/rigs`), with user overrides in `GenericDataLocation/wfview/rigs`.

**External control interfaces.** `rigctld.cpp` is wfview's own implementation of the Hamlib rigctl TCP protocol. `tciserver.cpp` implements the TCI websocket protocol, including TCI audio. `tcpserver.cpp` and `pttyhandler.cpp` provide raw CI-V passthrough over TCP and over a pseudo-terminal/virtual serial port. `usbcontroller.cpp` handles Shuttle/RC-28/gamepad controllers when `USB_CONTROLLER` is defined.

**Audio (`src/audio/`).** `audioHandlerBase` has a pair of input/output implementations for each backend: Qt Multimedia (`qt`), PortAudio (`pa`), RtAudio (`rt`) and TCI. `audioconverter` handles format conversion and Opus/ADPCM codecs. `rxaudioprocessor`/`txaudioprocessor` run the DSP chain. That chain includes bundled third-party code: speexdspmini, Audacity-derived noise reduction in `anr/`, the LADSPA-style plugins in `plugins/`, pocketfft and the resampler. These are vendored; avoid reformatting them.

**UI.** Widgets are Qt Designer `.ui` files in `src/`, each paired with a `.cpp` in `src/` and a `.h` in `include/`. `receiverwidget` draws the per-receiver spectrum/waterfall with QCustomPlot. Settings are handled in `settingswidget` and `prefs.h`. Log categories (`logRig()`, `logSystem()`, …) are declared in `logcategories.h`.

## Repo notes

- `old-source/` contains retired code that is not built.
- Translations are `translations/*.ts`, listed in `TRANSLATIONS` in `wfview.pro`.
- The version string defaults to the `WFVIEW_VERSION` define in `wfview.pro`. CI overrides it with `VERSION=` taken from git tags.
- `CI-V.md` is a reference table of Icom CI-V addresses.
