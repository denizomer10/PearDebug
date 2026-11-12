# PearDebug

Logging and diagnostics for the Pear stack:
[PearSilicon](https://github.com/denizomer10/PearSilicon),
[PearRuntime](https://github.com/denizomer10/PearRuntime),
[PearLegacyEngine](https://github.com/denizomer10/PearLegacyEngine),
Wine (MetalSharp Wine, WineHQ) and native macOS apps.

PearDebug ships as two parts:

- **`PearDebugKit`**: a Swift library. PearSilicon uses it for every launch log.
- **`peardebug`**: a command-line tool for reading, analyzing and probing.

## Modules

| Module | Purpose |
|---|---|
| `LogStore` | One log session per launch. The previous runs are rotated (`<id>.log`, `<id>.1.log` …), not truncated, so the log of a crash survives the relaunch. Large logs are trimmed. |
| `LogParser` | Turns PearRuntime (`[PearRuntime][area]`), PearLegacyEngine (`PearLegacyEngine::module:`), Wine (`0124:err:module:`), Rust panics and CRI middleware lines into structured events. |
| `Analyzer` | Rule modules per component (`PearLegacyEngine`, `PearRuntime`, `Wine`, `Common`). Each finding has a severity, the log lines it is based on, and a suggestion. Rules are plain Swift values, so a component can add its own. |
| `CrashReports` | Finds and summarizes macOS `.ips` crash reports for a process. |
| `LegacyProbe` | A GDB remote client for PearLegacyEngine's `--gdb` server: it watches guest functions by symbol name and reports their arguments, return values and callers. |
| `LegacyCrashCapture` | Runs a 32-bit game to its crash under the `--gdb` server and reads registers, the frame-pointer call chain and memory at the stack and at every register that points to memory. |
| `LegacyFrameDump` | Reads the engine's frame dumps (`PEAR_DUMP_FRAME`, `PEAR_PRESENT_DUMP_*`), writes PNGs and reports the render size and how much is drawn. |
| `WineProcesses` | Every process of a Wine prefix, whichever engine started it: matched by working directory and by the prefix's wineserver folder, because Wine ships no process list and rewrites process names. Also force-kills them. PearSilicon uses it for running state and Stop. |
| `SteamDiagnostics` | Explains a Windows Steam client's state from its own logs and the prefix registry: CEF path rule, StorageService fatal, drive-root install folder (repair loop), UI closed at start, sign-in window shown. |

## Command line

```sh
swift build -c release
.build/release/peardebug analyze com.rockstargames.gtavc.de      # newest log for an app
.build/release/peardebug analyze ./some.log --format markdown
.build/release/peardebug logs com.sega.MikuFlick --source legacy --follow
.build/release/peardebug list com.miHoYo.GenshinImpact
.build/release/peardebug crashes GenshinImpact
```

`analyze` exits with status 2 when it finds a problem, so it can gate scripts.

### Probing a 32-bit game

Start the engine with a GDB server and a time to stop at, then watch functions:

```sh
PEAR_BREAK_AFTER=215 touchHLE MikuFlick.app --gdb=127.0.0.1:9001 --no-error-popup &
peardebug legacy probe MikuFlick.app/MikuFlick --port 9001 \
  --watch criManaPlayer_GetStatus,criMvPly_IsEndMainAudioPlayback --count 20
peardebug legacy symbolize MikuFlick.app/MikuFlick 0x65798 0x77b67
```

### More tools

```sh
peardebug legacy crash --port 9001 --binary Game.app/Game     # engine started with --gdb=127.0.0.1:9001
peardebug legacy frames /tmp/mf/f --flip                      # after PEAR_DUMP_FRAME=/tmp/mf/f
peardebug wine ps "~/Library/Application Support/PearSilicon/wine-apps/<Game>"
peardebug steam "<entry>/Prefix/drive_c/Program Files (x86)/Steam"
```

See [DEBUGGING.md](DEBUGGING.md) for the full set of methods, with the
problems each one found.

## License

MIT. See [LICENSE](LICENSE).
