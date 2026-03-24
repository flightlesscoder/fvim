# 01 — Set Up C# Project

**Status:** pending

## Overview

Replace `fvim.fsproj` with `fvim.csproj`, configure the project for .NET 6.0 with the same output settings, and install all required NuGet packages. Remove all F#-specific packages (`FSharp.Core`, `FSharp.Data`, `FSharp.Control.Reactive`, `FSharp.Span.Utils`, `FSharp.SystemTextJson`) and replace them with C# equivalents.

## Motivation

F# projects use an ordered `<Compile>` list (source files must be declared in dependency order). C# projects use wildcards. The project also needs C# language version 11+ for `required` members, file-scoped namespaces, and switch expressions. The F# runtime (`FSharp.Core`) is not needed in C#.

## Steps

1. **Rename the project file**: Copy `fvim.fsproj` to `fvim.csproj`. Open it and replace the first line with:
   ```xml
   <Project Sdk="Microsoft.NET.Sdk">
   ```

2. **Update PropertyGroup**: Keep existing properties and add:
   ```xml
   <LangVersion>11</LangVersion>
   <Nullable>enable</Nullable>
   <ImplicitUsings>enable</ImplicitUsings>
   ```
   Remove `<Prefer32Bit>false</Prefer32Bit>` (not valid for C# SDK style).

3. **Replace the `<Compile>` ItemGroup**: Remove the entire ordered `<Compile Include="...">` list. C# projects use implicit includes. Add:
   ```xml
   <Compile Include="**\*.cs" Exclude="obj\**" />
   ```

4. **Keep XAML/resource ItemGroups unchanged**: The `<AvaloniaResource>`, `<Content>`, `<EmbeddedResource>`, `<ApplicationDefinition>`, and `<None>` items stay as-is.

5. **Remove F# packages** from the `<PackageReference>` list:
   - `FSharp.Core`
   - `FSharp.Data` (replaced by `System.Text.Json`)
   - `FSharp.Control.Reactive` (replaced by `System.Reactive`)
   - `FSharp.Span.Utils` (replaced by built-in span APIs)
   - `FSharp.SystemTextJson` (F#-specific, not needed)

6. **Add replacement packages**:
   ```xml
   <PackageReference Include="System.Reactive" Version="6.0.0" />
   ```
   All other packages (`Avalonia.*`, `MessagePack`, `Microsoft.Win32.Registry`, `UACHelper`, `NSubsys`, `XamlNameReferenceGenerator`) remain unchanged.

7. **Update `XamlNameReferenceGenerator`**: Ensure it is still listed. It works for both F# and C# Avalonia projects.

8. **Create `GlobalUsings.cs`** at the project root:
   ```csharp
   global using System;
   global using System.Collections.Generic;
   global using System.Threading.Tasks;
   global using Avalonia.Media;
   ```

9. **Delete `fvim.fsproj`** after verifying `fvim.csproj` builds (even with empty source files).

10. **Verify build**: Run `dotnet build` — it will fail due to missing source files but should not fail on project configuration errors.

## References

- Original project file: `fvim.fsproj`
- Avalonia C# template: https://github.com/AvaloniaUI/avalonia-dotnet-templates
- System.Reactive NuGet: replaces `FSharp.Control.Reactive`
