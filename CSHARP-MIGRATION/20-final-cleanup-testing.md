# 20 — Final Cleanup and Testing

**Status:** pending

## Overview

After all source files are translated, perform a full integration test pass: build, run, and manually test all major FVim features. Remove all `.fs` files, clean up leftover F#-isms, and verify the publish pipeline.

## Motivation

The C# codebase is functionally complete after tasks 01–19, but integration testing will reveal issues not caught by compilation alone: msgpack framing bugs, incorrect DU case translations, rendering regressions, and platform-specific behaviors.

## Steps

### Build Verification

1. **Run `dotnet build`** and resolve all compilation errors. Common issues after DU migration:
   - Missing `default` arms in `switch` expressions
   - Incorrect `record struct` vs `record class` choice causing value-type vs reference-type bugs
   - `null` vs `Option`/`ValueOption` confusion — F# `voption` (value option) maps to `T?` for value types

2. **Run `dotnet build --configuration Release`** to catch trimming-related issues.

3. **Fix any `CS8600` / `CS8602` nullable reference warnings** that indicate real null-safety bugs (not just annotation issues).

### Runtime Testing

4. **Test embedded Neovim launch** (`--serveropts Embedded`):
   - Start FVim normally, verify Neovim spawns and connects
   - Type text, switch modes, verify rendering

5. **Test remote Neovim connection** (`--server tcp://...` or named pipe):
   - Start Neovim separately: `nvim --listen 127.0.0.1:6666`
   - Connect FVim: `fvim --server 127.0.0.1:6666`

6. **Test FVR daemon** (`--fvim-daemon`):
   - Start daemon: `fvim --fvim-daemon`
   - Attach: `fvim --fvim-remote-tab`

7. **Test rendering**:
   - Syntax highlighting (multiple highlight groups per line)
   - Unicode / CJK characters (double-width)
   - Emoji rendering
   - Nerd font icons (ligatures, special characters)
   - Cursor shapes (block, horizontal, vertical)
   - Cursor blinking
   - Popup menu (`:` command completion)
   - Float windows (e.g. hover docs in LSP)
   - Scroll operations (verify no rendering artifacts)

8. **Test window operations**:
   - Custom title bar (Windows)
   - Acrylic/blur background composition (Windows 11)
   - Window resize, maximize, minimize
   - Multi-grid (`ui_multigrid`) — split windows

9. **Test input handling**:
   - Regular key input
   - Ctrl, Alt, Meta modifier keys
   - AltGr combinations (European keyboards)
   - Mouse clicks, drags, scroll wheel
   - `<Shift-Space>` behavior (`key_disableShiftSpace`)

10. **Test config persistence**:
    - Close FVim, reopen — verify window position/size restored
    - Verify `~/.local/share/fvim/config.json` (Linux) or `%APPDATA%\fvim\config.json` (Windows) is written correctly

11. **Test Windows shell integration** (Windows only):
    - Run `fvim --fvim-setup` and verify file associations in registry
    - Run `fvim --fvim-uninstall` and verify cleanup

### Code Cleanup

12. **Remove all `.fs` source files** after the build is green and runtime tests pass.

13. **Remove `FSharp.Core` update from PackageReference** if it was left in csproj.

14. **Remove the workaround `type Foo = A`** sentinel type from `states.fs` — already done if translated, but double-check.

15. **Clean up `#nowarn` comments** — search for F# pragma comments left in C# files.

16. **Review all `TODO` comments** inherited from the F# source (e.g. `// TODO need further investigation`, `// TODO ui-ext-options`, `//TODO` in `Rune.Codepoint` for `Composed`) and either implement or file issues.

### Known Gotchas to Verify

17. **`voption` (ValueOption)**: F# uses `int voption` in `GridCell` (hl_id, repeat). These were translated to `int?` — verify the calling code handles `null` correctly in all hot paths (cell rendering is called millions of times per second).

18. **`ReadOnlyMemory<GridLine>`**: The `GridLineCmd` uses `ReadOnlyMemory<GridLine>` to avoid allocation. Verify the C# translation preserves this — do not convert to `GridLine[]` in the hot path.

19. **`[<Struct>]` DU performance**: Several F# DUs were `[<Struct>]` (stack-allocated). The C# `sealed record` translations are heap-allocated. For `GridCell` (which fills a 2D buffer) consider `readonly struct` instead of `record class`.

20. **Thread safety**: F# modules are initialized once and their mutable state is accessed from both the UI thread and background tasks. In C# ensure that `States.*` fields are accessed consistently (add `lock` or use `volatile` where needed).

21. **`Observable.throttle` on OSX**: The F# `flush_throttle` skips throttling on macOS. Verify this is preserved in `Model.cs`.

22. **MessagePack integer types**: Neovim sends integers that may be `int64` in msgpack but `int32` in F# pattern matching. Verify `Integer32` / `TryGetInteger32` helpers correctly downcast 64-bit integers.

## References

- Source: All `.fs` files
- FVim README: `README.md` — lists all supported `--fvim-*` options to test
- Neovim RPC API: https://neovim.io/doc/user/api.html
