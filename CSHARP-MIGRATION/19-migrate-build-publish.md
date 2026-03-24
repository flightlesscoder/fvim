# 19 — Migrate Build System and Publish Configuration

**Status:** pending

## Overview

Update the project file, CI/CD pipeline (`azure-pipelines.yml`), and publish configuration to work with C# instead of F#. Remove F#-specific MSBuild properties, update the ordered `<Compile>` list to wildcard includes, and verify trimming/R2R still works.

## Motivation

F# requires an explicit ordered `<Compile>` item list because it compiles files in declaration order. C# projects use `<Compile Include="**\*.cs" />` (implicit wildcard). F#-specific packages must be removed. The `fvim_avalonia.fsproj` sub-project may need to become `fvim_avalonia.csproj` or be merged.

## Steps

1. **Finalize `fvim.csproj`** — verify these properties are present and correct:
   ```xml
   <PropertyGroup>
     <TargetFramework>net6.0</TargetFramework>
     <AssemblyName>FVim</AssemblyName>
     <OutputType>Exe</OutputType>
     <LangVersion>11</LangVersion>
     <Nullable>enable</Nullable>
     <ImplicitUsings>enable</ImplicitUsings>
     <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
     <ApplicationManifest>app.manifest</ApplicationManifest>
     <ApplicationIcon>Assets\fvim.ico</ApplicationIcon>
     <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
   </PropertyGroup>

   <PropertyGroup Condition="'$(Configuration)' == 'Release'">
     <OutputType>WinExe</OutputType>
     <PublishTrimmed>true</PublishTrimmed>
     <PublishReadyToRun>true</PublishReadyToRun>
     <TrimMode>link</TrimMode>
     <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
   </PropertyGroup>
   ```

2. **Remove F# packages from `<ItemGroup>`**:
   - `FSharp.Core`
   - `FSharp.Data`
   - `FSharp.Control.Reactive`
   - `FSharp.Span.Utils`
   - `FSharp.SystemTextJson`

3. **Verify remaining packages**:
   ```xml
   <PackageReference Include="Avalonia.Desktop" Version="0.10.19" />
   <PackageReference Include="Avalonia.ReactiveUI" Version="0.10.19" />
   <PackageReference Include="Avalonia.Diagnostics" Version="0.10.19" />
   <PackageReference Include="Avalonia.Svg" Version="0.10.18" />
   <PackageReference Include="XamlNameReferenceGenerator" Version="1.6.1" />
   <PackageReference Include="System.Reactive" Version="6.0.0" />
   <PackageReference Include="MessagePack" Version="2.5.108" />
   <PackageReference Include="Microsoft.Win32.Registry" Version="5.0.0" />
   <PackageReference Include="UACHelper" Version="1.3.0.5" />
   <PackageReference Include="NSubsys" Version="1.0.0" />
   ```

4. **Resolve `PublishTrimmed` compatibility**: MessagePack and ReactiveUI may need trim annotations. Add trim suppressors for known problematic assemblies:
   ```xml
   <ItemGroup>
     <TrimmerRootAssembly Include="Avalonia.Themes.Fluent" />
     <TrimmerRootAssembly Include="ReactiveUI" />
   </ItemGroup>
   ```

5. **Check `XamlNameReferenceGenerator`**: With C# source generators, this works the same. Ensure generated code is committed or `.gitignore` excludes `obj/`.

6. **Update `azure-pipelines.yml`**: The F# build likely uses `dotnet build fvim.fsproj`. Update to:
   ```yaml
   - script: dotnet build fvim.csproj --configuration Release
   - script: dotnet publish fvim.csproj --configuration Release --runtime win-x64 --self-contained
   ```
   Remove any F#-specific steps (e.g. F# compiler version pins).

7. **Check `Avalonia/fvim_avalonia.fsproj`**: If this file exists as a separate Avalonia-specific project, determine whether it is still needed. If it was a compatibility shim, it can likely be deleted; if it contains separate code, translate it to a `.csproj` as well.

8. **Delete all `.fs` files** after all tasks are complete and the build passes. Do not delete until the build is green.

9. **Run `dotnet publish` for all target runtimes** listed in the original CI:
   ```
   dotnet publish --runtime win-x64 --self-contained
   dotnet publish --runtime linux-x64 --self-contained
   dotnet publish --runtime osx-x64 --self-contained
   ```

10. **Verify embedded resources**: The Nerd font (`Fonts/nerd.ttf`) is embedded via:
    ```xml
    <EmbeddedResource Include="Fonts\nerd.ttf" />
    ```
    This is unchanged — ensure it still loads at runtime via `Assembly.GetExecutingAssembly().GetManifestResourceStream(...)`.

## References

- Source: `fvim.fsproj`, `azure-pipelines.yml`
- .NET trimming docs: https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/trimming-options
- ReadyToRun: https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run
