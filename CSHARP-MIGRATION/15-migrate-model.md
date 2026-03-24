# 15 — Migrate Core Model / Redraw Dispatcher (`model.fs`)

**Status:** pending

## Overview

Translate `model.fs` to `Model.cs`. This module is the central dispatcher that receives `RedrawCommand` values from Neovim and routes them to the appropriate grid/frame. It owns the `grids` and `frames` hashmaps, the `Nvim` instance, and the `Start()` / `StopAsync()` lifecycle.

## Motivation

`model.fs` uses a large `let rec redraw cmd = match cmd with ...` function — a recursive pattern match on the `RedrawCommand` DU. In C# this becomes a `switch` expression or `switch` statement. The module-level `let` bindings in a private `ModelImpl` module become a `private static` inner class or top-level private fields. The `#nowarn "0025"` pragma suppresses incomplete pattern warnings — in C# you must add a `default` arm to all switch expressions.

## Steps

1. **Create `Model.cs`** as `public static class Model`.

2. **Translate private state** (the `ModelImpl` private module):
   ```csharp
   private static readonly Nvim _nvim = new();
   private static readonly Subject<Unit> _evUiOpt = new();
   private static readonly Subject<Unit> _evFlush = new();
   private static readonly Dictionary<int, IGridUI> _grids = new();
   private static readonly Dictionary<int, IFrame> _frames = new();
   private static readonly TaskCompletionSource<Unit> _init = new();
   ```

3. **Translate `add_grid` / `destroy_grid`**:
   ```csharp
   private static void AddGrid(IGridUI grid) => _grids[grid.Id] = grid;
   private static void DestroyGrid(int id) {
       if (_grids.Remove(id, out var grid)) grid.Detach();
   }
   ```

4. **Translate `unicast` / `broadcast`**:
   ```csharp
   private static void Unicast(int id, RedrawCommand cmd) {
       if (_grids.TryGetValue(id, out var grid)) grid.Redraw(cmd);
       else Trace($"unicast into non-existing grid #{id}: {cmd}");
   }
   private static void Broadcast(RedrawCommand cmd) {
       foreach (var grid in _grids.Values) grid.Redraw(cmd);
   }
   ```

5. **Translate `unicast_detach` / `unicast_change_parent` / `unicast_create`**:
   These helpers handle grid lifecycle during `WinPos`, `WinFloatPos`, `WinClose` events. Translate directly.

6. **Translate the `redraw()` function** — the large recursive match. In C# use a `switch` statement (not expression, since it has side effects):
   ```csharp
   private static void Redraw(RedrawCommand cmd) {
       switch (cmd) {
           case UnknownCommandCmd(var x):
               Trace($"unknown command {x}"); break;
           case SetTitleCmd(var title):
               SetTitle(1, title); break;
           case SetIconCmd: break;
           case ModeInfoSetCmd(var csEnabled, var info):
               Theme.ModeDefs = info;
               Broadcast(cmd); break;
           case ModeChangeCmd(var name, var idx):
               Broadcast(cmd); break;
           case MouseCmd(var on):
               // set mouse state
               break;
           case BusyCmd(var on):
               Broadcast(cmd); break;
           case BellCmd:
               Bell(false); break;
           case VisualBellCmd:
               Bell(true); break;
           case FlushCmd:
               _evFlush.OnNext(Unit.Default);
               Broadcast(cmd); break;
           case DefaultColorsSetCmd:
               Broadcast(cmd); break;
           case HighlightAttrDefineCmd(var attrs):
               foreach (var a in attrs)
                   if (a.Id >= 0 && a.Id < Theme.HiDefs.Length)
                       Theme.HiDefs[a.Id] = a;
               Broadcast(cmd); break;
           case GridLineCmd:
               // unicast to specific grid
               // The grid id is in each GridLine — broadcast or unicast per grid
               break;
           case GridClearCmd(var grid):
               Unicast(grid, cmd); break;
           case GridDestroyCmd(var grid):
               DestroyGrid(grid); break;
           case GridCursorGotoCmd(var grid, _, _):
               Unicast(grid, cmd); break;
           case GridScrollCmd(var grid, _, _, _, _, _, _):
               Unicast(grid, cmd); break;
           case GridResizeCmd(var grid, var w, var h):
               UnicastCreate(grid, cmd, w, h); break;
           case WinPosCmd(var grid, _, _, _, var w, var h):
               UnicastCreate(grid, cmd, w, h); break;
           case WinFloatPosCmd(var grid, _, _, _, _, _, _, _):
               // change parent
               break;
           case WinHideCmd(var grid):
               Unicast(grid, cmd); break;
           case WinCloseCmd(var grid):
               UnicastDetach(grid, cmd); break;
           case WinScrollOverStartCmd:
           case WinScrollOverResetCmd:
               Broadcast(cmd); break;
           case WinViewportCmd(var grid, _, _, _, _, _, _):
               Unicast(grid, cmd); break;
           case WinExtmarksCmd(var grid, _, _):
               Unicast(grid, cmd); break;
           case MsgSetPosCmd(var grid, _, _, _):
               UnicastCreate(grid, cmd, 0, 0); break;
           case PopupMenuShowCmd:
           case PopupMenuSelectCmd:
           case PopupMenuHideCmd:
               Broadcast(cmd); break;
           case SetOptionCmd(var opts):
               foreach (var opt in opts) HandleUiOption(opt);
               break;
           case SemanticHighlightGroupSetCmd(var groups):
               Theme.Semhl = groups;
               break;
           case MultiRedrawCommandCmd(var cmds):
               foreach (var c in cmds) Redraw(c); break;
           default:
               Trace($"unhandled redraw command: {cmd}"); break;
       }
   }
   ```

7. **Translate `HandleUiOption(opt)`** — handles `UiOption` variants: font changes, linespace, tabline visibility, etc. This triggers events on `Theme` for font/theme updates.

8. **Translate `Start()` method** — the public entry point that connects to Neovim, sets up subscriptions, and initializes the UI:
   ```csharp
   public static async Task Start(Options opts, IFrame frame) {
       AddFrame(frame);
       AddGrid(frame.MainGrid);
       _nvim.Start(opts);
       // subscribe to nvim events
       _nvim.Events.Subscribe(OnEvent);
       // send ui_attach
       await _nvim.UiAttach(...);
       await _init.Task;
   }
   ```

9. **Translate `OnEvent(e)`** — dispatches `Event` variants:
   ```csharp
   private static void OnEvent(Event e) {
       switch (e) {
           case NotificationEvent { Req: { Method: "redraw", Parameters: var ps } }:
               // parse and dispatch redraw commands
               break;
           case RpcRequestEvent req:
               // handle RPC requests
               break;
           case CrashEvent { Code: var code }:
               // show crash dialog
               break;
           case ExitEvent { Code: var code }:
               // handle exit
               break;
       }
   }
   ```

10. **Translate `flush_throttle`** — the OS X vs Windows throttle:
    ```csharp
    private static IObservable<T> FlushThrottle<T>(IObservable<T> obs) =>
        RuntimeInformation.IsOSPlatform(OSPlatform.OSX)
            ? obs
            : obs.Throttle(TimeSpan.FromMilliseconds(10));
    ```

11. **Translate `Dispatcher.UIThread.Post`**: The F# code uses `Avalonia.Threading.Dispatcher.UIThread.Post(...)` for thread marshaling. Use the same in C#.

12. **`add_frame` / `setTitle`**:
    ```csharp
    private static void AddFrame(IFrame frame) => _frames[frame.MainGrid.Id] = frame;
    private static void SetTitle(int id, string title) {
        if (_frames.TryGetValue(id, out var frame)) frame.Title = title;
    }
    ```

## References

- Source: `model.fs`
- `#nowarn "0025"` = incomplete match warning. In C# you get a compile error if switch is non-exhaustive; always add `default` arm.
- Avalonia threading: `Dispatcher.UIThread.Post(action)` works identically in C#
