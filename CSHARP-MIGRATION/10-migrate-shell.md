# 10 — Migrate Windows Shell Integration (`shell.fs`)

**Status:** pending

## Overview

Translate `shell.fs` to `Shell.cs`. This file handles Windows-specific tasks: UAC elevation checking, registry-based file association setup, and teardown. It uses `Microsoft.Win32.Registry` and `UACHelper` NuGet packages — both of which are C# libraries used directly from F#.

## Motivation

`shell.fs` is almost entirely `System.Win32.Registry` API calls wrapped in F# functions. The translation is mechanical. The F# `task { }` computation expressions become `async Task`. The registry writes use `Registry.SetValue` / `Registry.CreateSubKey` which are identical in C#.

## Steps

1. **Create `Shell.cs`** as `public static class Shell`.

2. **Translate `win32CheckUAC()`**:
   ```csharp
   public static bool Win32CheckUAC() {
       // Uses UACHelper.UACHelper.IsElevated or WindowsIdentity
       var identity = System.Security.Principal.WindowsIdentity.GetCurrent();
       var principal = new System.Security.Principal.WindowsPrincipal(identity);
       return principal.IsInRole(System.Security.Principal.WindowsBuiltInRole.Administrator);
   }
   ```
   If using `UACHelper` package: `return UACHelper.UACHelper.IsElevated;`

3. **Translate `win32RegisterFileAssociation()`**:
   Read the original F# source to get the exact registry keys being written. The pattern is:
   ```csharp
   public static void Win32RegisterFileAssociation() {
       // HKCU\Software\Classes\.txt → "fvim"
       // HKCU\Software\Classes\fvim\shell\open\command → "fvim.exe %1"
       // etc.
       using var key = Registry.CurrentUser.CreateSubKey(@"Software\Classes\fvim");
       key.SetValue("", "FVim Document");
       // ... more registry writes
   }
   ```

4. **Translate `win32UnregisterFileAssociation()`**:
   ```csharp
   public static void Win32UnregisterFileAssociation() {
       Registry.CurrentUser.DeleteSubKeyTree(@"Software\Classes\fvim", throwOnMissingSubKey: false);
       // ... additional cleanup
   }
   ```

5. **Translate `setup()` / `uninstall()`**:
   ```csharp
   public static void Setup() {
       if (!Win32CheckUAC()) {
           // Relaunch elevated
           // UACHelper.UACHelper.RunElevated(...)
       }
       Win32RegisterFileAssociation();
   }

   public static void Uninstall() {
       Win32UnregisterFileAssociation();
   }
   ```

6. **Platform guard**: The F# code includes platform guards (`RuntimeInformation.IsOSPlatform(OSPlatform.Windows)`). Preserve these:
   ```csharp
   public static void Setup() {
       if (!RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
           throw new PlatformNotSupportedException("Shell integration is Windows-only.");
       // ...
   }
   ```

7. **Wrap registry calls in `try/catch`**: Registry operations can fail silently. Mirror the F# `try ... with _ -> ()` pattern with `try { } catch { }`.

8. **Keep NuGet packages** `Microsoft.Win32.Registry` and `UACHelper` — they are already C# libraries.

## References

- Source: `shell.fs`
- `Microsoft.Win32.Registry` API: https://learn.microsoft.com/en-us/dotnet/api/microsoft.win32.registry
- `UACHelper` package: https://github.com/falahati/UACHelper
