# 18 — Migrate Application Entry Point (`App.xaml.fs`, `Program.fs`)

**Status:** pending

## Overview

Translate `App.xaml.fs` and `Program.fs` to C#. These are the application root and entry point. `App.xaml.fs` defines the `App` class (Avalonia application root) and the `ViewLocator` (which maps ViewModels to Views). `Program.fs` is the `[<EntryPoint>]` function that bootstraps Avalonia and launches FVim.

## Motivation

F#'s `[<EntryPoint>]` attribute maps to C#'s `static void Main(string[] args)`. The `App` class pattern in Avalonia is nearly identical between F# and C# — the main difference is that F# uses `type App() = inherit Application()` while C# uses `class App : Application`. The `ViewLocator` uses reflection to find View types from ViewModel names.

## Steps

### Program.fs

1. **Create `Program.cs`**:
   ```csharp
   using Avalonia;
   using Avalonia.ReactiveUI;

   class Program {
       [STAThread]
       public static int Main(string[] args) {
           var opts = GetOpt.ParseOptions(args);

           // Handle non-GUI intents first
           if (opts.Intent == Intent.Setup) {
               Shell.Setup();
               return 0;
           }
           if (opts.Intent == Intent.Uninstall) {
               Shell.Uninstall();
               return 0;
           }
           if (opts.Intent == Intent.Daemon) {
               Daemon.RunDaemon(Daemon.DefaultDaemonName).Wait();
               return 0;
           }

           // Set up logging
           if (opts.LogToStdout)
               Log.LogStream.Subscribe(Console.Error.WriteLine);
           if (!string.IsNullOrEmpty(opts.LogToFile))
               Log.LogStream.Subscribe(msg => File.AppendAllText(opts.LogToFile, msg + "\n"));

           return BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);
       }

       public static AppBuilder BuildAvaloniaApp() =>
           AppBuilder.Configure<App>()
               .UsePlatformDetect()
               .WithInterFont()
               .LogToTrace()
               .UseReactiveUI();
   }
   ```

2. **Handle `--fvim-no-fork`**: The F# version has platform-specific fork handling on Linux/macOS. Check the original for the exact flag and replicate.

3. **AppDomain unhandled exception handler**: The F# program.fs hooks `AppDomain.CurrentDomain.UnhandledException`. Replicate:
   ```csharp
   AppDomain.CurrentDomain.UnhandledException += (_, e) => {
       // push to nvim event stream as UnhandledException event
   };
   ```

### App.xaml.fs

4. **Create `App.xaml.cs`**:
   ```csharp
   public partial class App : Application {
       public override void Initialize() {
           AvaloniaXamlLoader.Load(this);
       }

       public override void OnFrameworkInitializationCompleted() {
           if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop) {
               desktop.MainWindow = new Frame {
                   DataContext = new FrameViewModel()
               };
               // Start the model
               _ = Model.Start(/* opts */, (FrameViewModel)desktop.MainWindow.DataContext!);
           }
           base.OnFrameworkInitializationCompleted();
       }
   }
   ```

5. **Translate `ViewLocator`**:
   ```csharp
   public class ViewLocator : IDataTemplate {
       public bool Match(object? data) => data is ViewModelBase;

       public Control Build(object? data) {
           var name = data!.GetType().FullName!
               .Replace("ViewModel", "")
               .Replace("FVim.", "FVim.Views.");
           var type = Type.GetType(name);
           if (type is not null)
               return (Control)Activator.CreateInstance(type)!;
           return new TextBlock { Text = $"Not Found: {name}" };
       }
   }
   ```

6. **Register `ViewLocator` in `App.xaml`**: The XAML file should already have this via `<Application.DataTemplates>`. No change needed if the class name stays the same.

7. **DesignTime data** — the `DesignTime/` folder contains sample data for Avalonia Designer. Translate these to C# as well:
   - `DesignTime/TitleBarSampleData.fs` → `DesignTime/TitleBarSampleData.cs`
   - `DesignTime/GridSampleData.fs` → `DesignTime/GridSampleData.cs`
   - `DesignTime/CrashReportSampleData.fs` → `DesignTime/CrashReportSampleData.cs`
   - `DesignTime/MainWindowSampleData.fs` → `DesignTime/MainWindowSampleData.cs`
   These provide design-time DataContext values for the Avalonia Visual Designer.

8. **`app.manifest`**: Keep as-is — this is already XML and does not change.

9. **`fvim.config`**: Keep as-is — this is the `ApplicationDefinition` file and does not change.

## References

- Source: `App.xaml.fs`, `Program.fs`
- Avalonia `AppBuilder`: https://docs.avaloniaui.net/docs/next/concepts/application-lifetimes
- `IDataTemplate` / ViewLocator: https://docs.avaloniaui.net/docs/next/concepts/view-locator
- ReactiveUI + Avalonia integration: https://www.reactiveui.net/docs/getting-started/installation/avalonia
