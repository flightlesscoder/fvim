# 06 — Migrate Logging (`log.fs`)

**Status:** pending

## Overview

Translate `log.fs` to `Log.cs`. The F# module uses `FSharp.Control.Reactive` observables (`Subject<string>`, `Observable.merge`) for log routing, and printf-style format strings for `trace` and `error`. Replace with `System.Reactive` and `IObservable<string>`.

## Motivation

`FSharp.Control.Reactive` is a thin F# wrapper over `System.Reactive`. All of its functionality is directly available in `System.Reactive` which is idiomatic C#. The printf-style logging (e.g. `trace "neovim.process" fmt`) must be replaced with string interpolation or `string.Format`.

## Steps

1. **Add `System.Reactive` NuGet package** to `fvim.csproj` (already added in task 01).

2. **Create `Log.cs`** as a static class:

   ```csharp
   using System.Reactive.Linq;
   using System.Reactive.Subjects;

   public static class Log {
       private static readonly Subject<string> _traceSubject = new();
       private static readonly Subject<string> _errorSubject = new();

       public static IObservable<string> LogStream =>
           Observable.Merge(_traceSubject, _errorSubject);

       public static void Trace(string category, string message) =>
           _traceSubject.OnNext($"[{category}] {message}");

       public static void Error(string category, string message) =>
           _errorSubject.OnNext($"[ERROR:{category}] {message}");
   }
   ```

3. **Replace per-module `trace` helpers**: In each F# module, the pattern is:
   ```fsharp
   let inline private trace fmt = trace "neovim.process" fmt
   ```
   In C#, give each class a local helper:
   ```csharp
   private static void Trace(string fmt) => Log.Trace("neovim.process", fmt);
   ```
   Or use `ILogger<T>` from `Microsoft.Extensions.Logging` for a more idiomatic approach (optional upgrade, not required for direct migration).

4. **Translate `_logsSink`** (the merged observable exposed for consumption):
   The F# module exposes `_logsSink` as a merged observable subscribed to by the UI (crash reporter) and file/stdout sinks. In C#:
   ```csharp
   public static IObservable<string> LogStream { get; } = ...
   ```
   Subscribe from the startup code (`Program.cs` equivalent).

5. **Log patterns filtering**: The F# module supports filtering log messages by pattern. Translate:
   ```csharp
   public static IObservable<string> FilteredStream(IEnumerable<string> patterns) {
       if (!patterns.Any()) return LogStream;
       return LogStream.Where(msg => patterns.Any(p => msg.Contains(p)));
   }
   ```

6. **Stdout/file sinks**: Connect in `Program.cs`:
   ```csharp
   if (opts.LogToStdout)
       Log.LogStream.Subscribe(Console.WriteLine);
   if (!string.IsNullOrEmpty(opts.LogToFile))
       Log.LogStream.Subscribe(msg => File.AppendAllText(opts.LogToFile, msg + "\n"));
   ```

7. **Printf-style format strings**: F# uses `trace "category" "format %A %d" x y`. C# uses string interpolation. Replace all usages:
   - `trace "model" "unknown command %A" x` → `Trace($"unknown command {x}")`
   - `trace "neovim.process" "Starting process. Program: %s; Arguments: %A" prog args` → `Trace($"Starting process. Program: {prog}; Arguments: {args}")`

## References

- Source: `log.fs`
- `System.Reactive` docs: https://github.com/dotnet/reactive
- F# printf vs C# interpolation: use `$"..."` everywhere
