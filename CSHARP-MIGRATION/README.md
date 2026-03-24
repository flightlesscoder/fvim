# FVim — C# Migration Plan

This directory contains a structured migration plan for converting FVim from F# to C#. FVim is a cross-platform Neovim front-end built with Avalonia UI and ReactiveUI, targeting .NET 6.0.

## Task Index

Execute tasks in the order listed below. Each task has a `**Status:**` field at the top — update it to `in-progress` when starting and `done` when complete.

| # | File | Description |
|---|------|-------------|
| 01 | [01-setup-csharp-project.md](01-setup-csharp-project.md) | Replace `fvim.fsproj` with `fvim.csproj`, remove F# packages, add `System.Reactive` |
| 02 | [02-migrate-common-utilities.md](02-migrate-common-utilities.md) | Translate `common.fs` — type aliases, custom operators, active patterns, stream helpers |
| 03 | [03-migrate-type-definitions.md](03-migrate-type-definitions.md) | Translate `def.fs` — all DUs, struct records, `RedrawCommand` (30+ cases), parsing helpers |
| 04 | [04-migrate-config.md](04-migrate-config.md) | Replace `FSharp.Data.JsonProvider` with `System.Text.Json` POCOs |
| 05 | [05-migrate-getopt.md](05-migrate-getopt.md) | Translate `getopt.fs` — command-line parsing, connection option DUs |
| 06 | [06-migrate-logging.md](06-migrate-logging.md) | Translate `log.fs` — observable-based logging, replace `FSharp.Control.Reactive` |
| 07 | [07-migrate-states.md](07-migrate-states.md) | Translate `states.fs` — global mutable state → static class, `Set` → `HashSet` |
| 08 | [08-migrate-msgpack.md](08-migrate-msgpack.md) | Translate `msgpack.fs` — custom MessagePack formatter and resolver |
| 09 | [09-migrate-daemon.md](09-migrate-daemon.md) | Translate `daemon.fs` — FVR multiplexer, named pipe server, `task {}` → `async Task` |
| 10 | [10-migrate-shell.md](10-migrate-shell.md) | Translate `shell.fs` — Windows UAC and registry file association helpers |
| 11 | [11-migrate-neovim-rpc.md](11-migrate-neovim-rpc.md) | Translate `neovim.fs` — Neovim RPC protocol, `Nvim` class, msgpack read loop |
| 12 | [12-migrate-ui-theme-svg.md](12-migrate-ui-theme-svg.md) | Translate `ui.fs`, `theme.fs`, `svg.fs` — interfaces, font cache, highlight system |
| 13 | [13-migrate-input.md](13-migrate-input.md) | Translate `input.fs` — key/mouse mapping to Neovim notation, active patterns |
| 14 | [14-migrate-widgets-wcwidth.md](14-migrate-widgets-wcwidth.md) | Translate `widgets.fs`, `wcwidth.fs` — Unicode width tables, widget layout |
| 15 | [15-migrate-model.md](15-migrate-model.md) | Translate `model.fs` — central redraw dispatcher, grid/frame routing |
| 16 | [16-migrate-viewmodels.md](16-migrate-viewmodels.md) | Translate all ViewModels — ReactiveUI MVVM classes, `GridViewModel` (most complex) |
| 17 | [17-migrate-views.md](17-migrate-views.md) | Translate XAML code-behind — custom SkiaSharp rendering in `Grid.xaml.cs` |
| 18 | [18-migrate-app-program.md](18-migrate-app-program.md) | Translate `App.xaml.fs`, `Program.fs` — entry point, ViewLocator, DesignTime data |
| 19 | [19-migrate-build-publish.md](19-migrate-build-publish.md) | Update CI pipeline, publish config, trimming/R2R compatibility |
| 20 | [20-final-cleanup-testing.md](20-final-cleanup-testing.md) | Integration testing, delete `.fs` files, fix runtime regressions |

## Key F# → C# Mapping Reference

| F# Concept | C# Equivalent |
|------------|---------------|
| `[<Struct>]` DU (simple) | `enum` |
| DU with data (complex) | `abstract record` + `sealed record` subclasses |
| `[<Struct>]` record | `readonly record struct` |
| Active pattern `(|Foo|_|)` | `static bool TryMatchFoo(...)` method |
| `let mutable x = v` (module) | `public static T X = v` in `static class` |
| `type hashmap<'a,'b> = Dictionary<'a,'b>` | `Dictionary<K,V>` directly |
| `(>>=)` option bind | `?.` null-conditional or `Bind()` extension |
| `task { }` / `backgroundTask { }` | `async Task` / `Task.Run(async () => ...)` |
| `Event<T>()` + `.Trigger` | `Subject<T>` + `.OnNext()` |
| `Observable.merge` | `Observable.Merge(...)` |
| `FSharp.Control.Reactive` | `System.Reactive` |
| `FSharp.Data.JsonProvider` | `System.Text.Json` + hand-written POCOs |
| `Set.empty<string>` | `HashSet<string>` |
| `Map<K,V>` | `Dictionary<K,V>` |
| `int voption` | `int?` (nullable value type) |
| `#nowarn "0025"` (incomplete match) | `default:` arm in `switch` |
| `[<AutoOpen>]` module | `static class` with `using static` |
| `[<Literal>] let x = "y"` | `const string X = "y"` |
| `module private Foo = ...` | `private static class Foo { ... }` |
| `ReactiveObject` (F#) | `ReactiveObject` (identical in C#) |

## Architecture Overview

```
Program.cs         → Entry point, Avalonia bootstrap
App.xaml.cs        → Application root, ViewLocator
  └─ Model.cs      → Central dispatcher (nvim events → grid/frame routing)
      ├─ Neovim.cs → RPC protocol (msgpack read loop, call/notify)
      ├─ Daemon.cs → FVR multiplexer (named pipe server)
      └─ ...
  └─ ViewModels/
      ├─ FrameViewModel.cs  → Main window state (implements IFrame)
      └─ GridViewModel.cs   → Grid state + IGridUI (implements IGridUI)
  └─ Views/
      ├─ Frame.xaml.cs      → Main window
      └─ Grid.xaml.cs       → Custom SkiaSharp cell rendering
```
