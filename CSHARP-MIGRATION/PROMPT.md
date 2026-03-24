# FVim C# Migration — Executor Prompt

## Project Context

You are working on **FVim** — a cross-platform Neovim front-end GUI written in **F#** using the **Avalonia** UI framework. The project lives at `C:\projects\fvim\`. Your task is to migrate the entire codebase from F# to C#, following the structured task plan in this directory.

### What FVim Does
FVim communicates with a Neovim process (or remote session) via the **MessagePack RPC** protocol, then renders the resulting grid of characters using **SkiaSharp** on an Avalonia canvas. It supports:
- Local, WSL, SSH, and remote Neovim sessions
- Multi-grid (split windows, floating windows)
- A multiplexer daemon (FVR) for sharing Neovim sessions across windows
- Windows-specific features: Acrylic/blur composition, UAC, file association

### Repository Structure
```
C:\projects\fvim\
├── fvim.fsproj           # F# project file (to be replaced with fvim.csproj)
├── Program.fs            # Entry point
├── App.xaml.fs           # Avalonia App class + ViewLocator
├── common.fs             # Utility functions, type aliases, operators
├── config.fs             # JSON config (uses FSharp.Data.JsonProvider)
├── getopt.fs             # Command-line parsing
├── log.fs                # Observable-based logging
├── def.fs                # ALL core type definitions (DUs, records, parsers)
├── wcwidth.fs            # Unicode character width tables
├── daemon.fs             # FVR multiplexer server
├── shell.fs              # Windows UAC/registry integration
├── states.fs             # Global mutable state
├── msgpack.fs            # MessagePack custom formatter
├── neovim.fs             # Neovim RPC protocol layer
├── ui.fs                 # UI abstractions (IGridUI, IFrame, font cache)
├── svg.fs                # SVG icon rendering
├── theme.fs              # Highlight/color system
├── input.fs              # Keyboard/mouse → Vim notation mapping
├── widgets.fs            # Grid widget placement
├── model.fs              # Central redraw dispatcher
├── ViewModels/           # MVVM ViewModels (ReactiveUI)
├── Views/                # Avalonia Views + XAML code-behind
└── CSHARP-MIGRATION/     # ← YOU ARE HERE
    ├── README.md         # Task index
    ├── PROMPT.md         # This file
    └── 01-*.md ... 20-*.md  # Individual task files
```

## Your Mission

Execute the migration tasks **in order**, from `01-setup-csharp-project.md` through `20-final-cleanup-testing.md`.

### Workflow for Each Task

1. **Read the task file** to understand what needs to be done.
2. **Read the referenced F# source files** to understand the exact code being migrated — do not guess based on task descriptions alone.
3. **Update the task's `**Status:**`** from `pending` to `in-progress`.
4. **Implement the migration** — create C# files, update project files, etc.
5. **Verify the build compiles** with `dotnet build` before marking done.
6. **Update the task's `**Status:**`** to `done`.
7. **Commit the change**: `git add -A && git commit -m "migrate: <task name>"`
8. Move to the next task.

### Important Rules

- **Always read the original `.fs` file** before writing C#. The task files describe *what* to do; the actual code tells you *how*.
- **Do not delete `.fs` files** until task 20. The F# files are the source of truth during migration.
- **One commit per task** — keep history clean.
- **Run `dotnet build` after each task** to catch compilation errors early.
- **Do not try to run the app** until task 20 — it won't link correctly until all files are migrated.

## Critical F# Patterns to Handle Carefully

### 1. Discriminated Unions (DUs) — `def.fs`
The `RedrawCommand` DU has 30+ cases and is the central data structure. **Every case must be translated** — missing cases will cause silent routing failures.

Pattern in F#:
```fsharp
type RedrawCommand =
| SetTitle of string
| GridLine of ReadOnlyMemory<GridLine>
// ... 28 more
```

C# translation:
```csharp
public abstract record RedrawCommand;
public sealed record SetTitleCmd(string Title) : RedrawCommand;
public sealed record GridLineCmd(ReadOnlyMemory<GridLine> Lines) : RedrawCommand;
```

In `model.fs`, the `redraw` function exhaustively matches all cases. In C#, make sure the `switch` in `Model.Redraw()` has a `default` arm — the compiler will not catch missing cases on `abstract record` hierarchies unless you add analyzers.

### 2. Active Patterns — used throughout `def.fs`, `common.fs`, `input.fs`
F# active patterns like `(|Integer32|_|)` look like pattern match cases but are really functions. Example:

```fsharp
// Definition
let (|Integer32|_|) (x:obj) =
    match x with | :? int32 as x -> Some(int32 x) | _ -> None

// Usage in a match
match x with | Integer32 n -> use n | _ -> ...
```

C# translation — define as `static bool TryGetInteger32(object x, out int result)` and replace usages with `if (TryGetInteger32(x, out var n)) { use n }`.

**Watch out for nested active patterns** like `| ObjArray [| String cmd; Integer32 n |] ->` — these need nested `TryGet*` calls in C#.

### 3. F# Modules → Static Classes
F# `module FVim.model` with `let` bindings = C# `static class Model`. Module-private `let` bindings become `private static` members.

The F# pattern:
```fsharp
module FVim.model
// private module containing implementation
module private ModelImpl =
    let nvim = Nvim()
    let grids = hashmap[]
```

C# equivalent:
```csharp
public static class Model {
    private static readonly Nvim _nvim = new();
    private static readonly Dictionary<int, IGridUI> _grids = new();
}
```

### 4. `task { }` / `backgroundTask { }` → `async Task`
F# computation expressions map directly:
```fsharp
task {
    let! n = stream.ReadAsync(buf)
    if n = 0 then failwith "read"
    return n
}
```
→
```csharp
async Task<int> ReadAsync(Stream stream, Memory<byte> buf) {
    int n = await stream.ReadAsync(buf);
    if (n == 0) throw new IOException("read");
    return n;
}
```

`backgroundTask { }` → `Task.Run(async () => { ... })` to ensure thread-pool execution.

### 5. Observable Events — `FSharp.Control.Reactive` → `System.Reactive`
F# `Event<T>()` + `.Publish` + `.Trigger` → C# `Subject<T>` + `.OnNext`:

```fsharp
let ev_flush = Event<unit>()
ev_flush.Trigger()           // fire
ev_flush.Publish             // IObservable<unit>
```
→
```csharp
private static readonly Subject<Unit> _evFlush = new();
_evFlush.OnNext(Unit.Default);   // fire
_evFlush.AsObservable()          // IObservable<Unit>
```

`FSharp.Control.Reactive`'s `Observable.*` operators are replaced by `System.Reactive.Linq.Observable.*` — the method names are nearly identical (`Observable.merge` → `Observable.Merge`, `Observable.throttle` → `Observable.Throttle`).

### 6. `int voption` → `int?`
F# `voption` (value option) is used in `GridCell.hl_id` and `GridCell.repeat`. It's a struct (no heap allocation). In C#, `int?` (nullable int) is the equivalent. This is critical in hot paths — `GridCell` fills a 2D array that may have millions of entries.

If performance matters, consider `readonly record struct GridCell(Rune Text, int HlId, int Repeat)` where -1 means "no value" for `HlId` and `Repeat`.

### 7. `FSharp.Data.JsonProvider` → `System.Text.Json`
The `config.fs` type `ConfigObject.Root` is a **generated** type from a JSON sample. It does not exist as a real class — it's created at compile time by the type provider. You must hand-write the C# POCOs (see task 04).

The save/load logic must match the JSON schema exactly, as existing `~/.local/share/fvim/config.json` files on users' machines must continue to load correctly.

### 8. `Set<string>` → `HashSet<string>`
`states.fs` uses F#'s immutable `Set<string>` for `ui_available_opts`. In C# use `HashSet<string>` — it's mutable and O(1) lookup vs O(log n) for F# Set. The semantics are equivalent for this usage (add + contains).

### 9. `#nowarn "0025"` — Incomplete Pattern Match Warnings
F# `#nowarn "0025"` suppresses "incomplete pattern match" warnings in files that intentionally use partial matches (e.g., `GridViewModel.fs`). In C# there is no equivalent suppression — you must add `default:` arms. **Do not silently ignore unknown cases** — log them with `Trace` just like the F# code does.

### 10. `[<Struct>]` DU Performance
Several F# DUs use `[<Struct>]` to avoid heap allocation. The most important is `GridCell` which is stored in a large 2D array. Use `readonly record struct` in C# to preserve value semantics:
```csharp
public readonly record struct GridCell(Rune Text, int? HlId, int? Repeat);
```
Note: `Rune` is a `class` so `GridCell` will still have a reference — but the struct wrapper avoids one level of boxing.

## Packages to Remove vs Keep

**Remove (F#-only):**
- `FSharp.Core`
- `FSharp.Data` (JsonProvider)
- `FSharp.Control.Reactive`
- `FSharp.Span.Utils`
- `FSharp.SystemTextJson`

**Add:**
- `System.Reactive` v6.0.0

**Keep unchanged:**
- `Avalonia.Desktop`, `Avalonia.ReactiveUI`, `Avalonia.Diagnostics`, `Avalonia.Svg`
- `XamlNameReferenceGenerator`
- `MessagePack`
- `Microsoft.Win32.Registry`
- `UACHelper`
- `NSubsys`

## Getting Started

```bash
cd C:\projects\fvim

# Read the first task
cat CSHARP-MIGRATION/01-setup-csharp-project.md

# Begin execution
# ... follow the workflow above for each task
```

Good luck.
