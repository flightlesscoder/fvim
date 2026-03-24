# 04 — Migrate Configuration (`config.fs`)

**Status:** pending

## Overview

Translate `config.fs` to `Config.cs`. This file uses `FSharp.Data.JsonProvider<>` — a compile-time type provider that generates typed JSON accessors from a sample JSON literal. C# has no type providers; replace with `System.Text.Json` and POCOs.

## Motivation

`FSharp.Data.JsonProvider` is an F#-only feature (compile-time metaprogramming). The generated types (`ConfigObject.Root`, `ConfigObject.Workspace`, `ConfigObject.Mainwin`, `ConfigObject.Logging`, `ConfigObject.Default`) must be hand-written as C# classes annotated for JSON serialization.

## Steps

1. **Create C# model classes** that match the JSON schema shown in the `sample_config` literal in `config.fs`:
   ```csharp
   public class ConfigRoot {
       public ConfigWorkspace[] Workspace { get; set; } = Array.Empty<ConfigWorkspace>();
       public ConfigLogging Logging { get; set; } = new();
       public ConfigDefault? Default { get; set; }
   }
   public class ConfigWorkspace {
       public string Path { get; set; } = "";
       public ConfigMainwin Mainwin { get; set; } = new();
   }
   public class ConfigMainwin {
       public int X { get; set; }
       public int Y { get; set; }
       public int W { get; set; }
       public int H { get; set; }
       public string State { get; set; } = "Normal";
       public string? BackgroundComposition { get; set; }
       public bool? CustomTitleBar { get; set; }
       public bool? NoTitleBar { get; set; }
   }
   public class ConfigLogging {
       public bool EchoOnConsole { get; set; }
       public string LogToFile { get; set; } = "";
   }
   public class ConfigDefault {
       public int W { get; set; } = 800;
       public int H { get; set; } = 600;
   }
   ```

2. **Add JSON serializer options** (case-insensitive to handle PascalCase/camelCase in existing config files):
   ```csharp
   private static readonly JsonSerializerOptions _opts = new() {
       PropertyNameCaseInsensitive = true,
       WriteIndented = false,
   };
   ```

3. **Translate `load()`**:
   ```csharp
   public static ConfigRoot Load() {
       try {
           var json = File.ReadAllText(ConfigFile);
           return JsonSerializer.Deserialize<ConfigRoot>(json, _opts) ?? new ConfigRoot();
       } catch {
           return new ConfigRoot();
       }
   }
   ```

4. **Translate `save()`** — the F# version merges by CWD path:
   ```csharp
   public static void Save(ConfigRoot cfg, int x, int y, int w, int h,
       int defW, int defH, string state, string composition,
       bool customTitleBar, bool noTitleBar) {
       var cwd = Path.GetFullPath(Environment.CurrentDirectory);
       var ws = new ConfigWorkspace {
           Path = cwd,
           Mainwin = new ConfigMainwin { X=x, Y=y, W=w, H=h, State=state,
               BackgroundComposition=composition, CustomTitleBar=customTitleBar, NoTitleBar=noTitleBar }
       };
       var dict = cfg.Workspace.ToDictionary(w => w.Path, w => w);
       dict[cwd] = ws;
       var result = new ConfigRoot {
           Workspace = dict.Values.ToArray(),
           Logging = cfg.Logging,
           Default = new ConfigDefault { W = defW, H = defH }
       };
       try { File.WriteAllText(ConfigFile, JsonSerializer.Serialize(result, _opts)); }
       catch { }
   }
   ```

5. **Translate `configroot`, `configdir`, `configfile` path constants**:
   ```csharp
   public static readonly string ConfigRoot = RuntimeInformation.IsOSPlatform(OSPlatform.Windows)
       ? Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData)
       : Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
   public static readonly string ConfigDir = Path.Combine(ConfigRoot, "fvim");
   public static readonly string ConfigFile = Path.Combine(ConfigDir, "config.json");
   ```

6. **Replace module-level side effect** — `config.fs` runs `Directory.CreateDirectory(configdir)` at module init. Do this in a static constructor:
   ```csharp
   static Config() {
       try { Directory.CreateDirectory(ConfigDir); } catch { }
   }
   ```

7. **Remove `FSharp.Data` NuGet package** from `fvim.csproj`.

8. **Add `System.Text.Json`** (already part of .NET 6.0, no extra package needed).

9. **Update all call sites** that use `ConfigObject.Root`, `ConfigObject.Workspace`, etc. to use the new C# types. Search for `config.load()` and `config.save(` across the codebase.

## References

- Source: `config.fs`
- `FSharp.Data.JsonProvider` docs: https://fsprojects.github.io/FSharp.Data/library/JsonProvider.html
- `System.Text.Json` docs: https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview
