# 17 — Migrate Views (XAML Code-Behind)

**Status:** pending

## Overview

Translate the XAML code-behind files from F# to C#. The XAML files themselves (`.xaml`) do not change — only the code-behind `.fs` → `.cs` files change. Views include:
- `Views/ViewBase.fs` → `Views/ViewBase.cs`
- `Views/Grid.xaml.fs` → `Views/Grid.xaml.cs` (most complex — custom rendering)
- `Views/Frame.xaml.fs` → `Views/Frame.xaml.cs`
- `Views/Cursor.xaml.fs` → `Views/Cursor.xaml.cs`
- `Views/TitleBar.xaml.fs` → `Views/TitleBar.xaml.cs`
- `Views/PopupMenu.xaml.fs` → `Views/PopupMenu.xaml.cs`
- `Views/CompletionItem.xaml.fs` → `Views/CompletionItem.xaml.cs`
- `Views/CrashReport.xaml.fs` → `Views/CrashReport.xaml.cs`

## Motivation

F# code-behind for Avalonia views uses `inherit UserControl()` or `inherit Window()`. In C# these become `public partial class Foo : UserControl`. The code-behind connects the ViewModel, handles Avalonia's `OnAttachedToVisualTree` / `OnDetachedFromVisualTree`, and does custom rendering in `Grid.xaml.fs` (the most complex view).

## Steps

### ViewBase.fs

1. **Create `Views/ViewBase.cs`**:
   ```csharp
   public class ViewBase<T> : UserControl where T : class {
       protected T? ViewModel => DataContext as T;

       protected override void OnDataContextChanged(EventArgs e) {
           base.OnDataContextChanged(e);
           if (DataContext is T vm) Connect(vm);
       }

       protected virtual void Connect(T vm) { }
   }
   ```
   Or use the ReactiveUI `ReactiveUserControl<TViewModel>` base class which provides `this.ViewModel` automatically.

### Grid.xaml.fs (Most Complex)

2. **Create `Views/Grid.xaml.cs`** — this is a `UserControl` (or `Canvas`) with custom Avalonia rendering. The F# version:
   - Overrides `Render(DrawingContext)` (or subscribes to Avalonia's render invalidation)
   - Uses SkiaSharp via `DrawingContext.PlatformImpl` to get the `SKCanvas`
   - Renders text cells using `SKPaint`, `SKTypeface`, and `SKFont`
   - Handles ligatures by grouping consecutive cells with the same style
   - Draws cursors, underlines, undercurls

   In C#:
   ```csharp
   public partial class Grid : UserControl {
       private GridViewModel? _vm;

       protected override void OnDataContextChanged(EventArgs e) {
           base.OnDataContextChanged(e);
           _vm = DataContext as GridViewModel;
       }

       public override void Render(DrawingContext ctx) {
           if (_vm is null) return;
           // Get SkiaSharp surface from Avalonia context
           using var lease = ctx.PlatformImpl as ISkiaDrawingContextImpl;
           var canvas = lease?.SkCanvas;
           if (canvas is null) { base.Render(ctx); return; }
           RenderGrid(canvas, _vm);
       }

       private void RenderGrid(SKCanvas canvas, GridViewModel vm) {
           // For each draw operation in vm._drawOps:
           // - Scroll: use canvas.Save/Restore + canvas.Translate
           // - Put: render cells in the dirty rect
           // ...
       }
   }
   ```

3. **Translate the cell rendering pipeline** from `Grid.xaml.fs`:
   - Group cells by highlight ID for batch rendering
   - Apply font (normal/bold/italic/wide/emoji) from `UI.GetTypeface`
   - Render background rectangles first, then text, then decorations
   - Handle zero-width and double-width characters (use `WcWidth.GetWidthAttr`)
   - Render cursor overlay last

4. **Translate scroll rendering**: The F# version uses a `Scroll` draw op to shift existing rendered content. In C#:
   ```csharp
   private void RenderScroll(SKCanvas canvas, ScrollOp op, double glyphW, double glyphH) {
       var src = new SKRect(...); // source rect in pixels
       var dst = new SKRect(...); // destination rect in pixels
       // Use canvas.DrawBitmap or copy region
   }
   ```

### Frame.xaml.fs

5. **Create `Views/Frame.xaml.cs`** — the main `Window`. Handles:
   - Window title binding
   - Background image/composition (Acrylic/Blur on Windows/macOS)
   - Custom title bar (when `CustomTitleBar = true`)
   - Window position/size save on close
   ```csharp
   public partial class Frame : Window {
       protected override void OnClosing(CancelEventArgs e) {
           // save window position to config
           if (DataContext is FrameViewModel vm) {
               Config.Save(Config.Load(), (int)Position.X, (int)Position.Y, ...);
           }
           base.OnClosing(e);
       }
   }
   ```

### Cursor.xaml.fs

6. **Create `Views/Cursor.xaml.cs`**: Renders the cursor shape. Subscribes to `CursorViewModel` properties. Typically a `Control` that draws a rectangle/underline/beam.

### Remaining Views

7. **`TitleBar.xaml.cs`**: Custom title bar with window controls (minimize/maximize/close buttons). Uses `Window.ExtendClientAreaToDecorationsHint`.

8. **`PopupMenu.xaml.cs`**: The completion popup. Binds to `PopupMenuViewModel.Items`. Uses a `ListBox` or `ItemsControl`.

9. **`CompletionItem.xaml.cs`**: Per-item view in the popup menu.

10. **`CrashReport.xaml.cs`**: Simple dialog showing exception details.

### XAML File Updates

11. **Update all `.xaml` files** to reference C# code-behind:
    - Change `x:Class="FVim.Views.Grid"` — this already works as-is since the class name stays the same
    - Ensure code-behind files use `partial class` matching the XAML `x:Class`

12. **`fvim.csproj` XAML settings**: The existing `<Compile Update="**\*.xaml.fs">` item must become `<Compile Update="**\*.xaml.cs">`. Update accordingly.

## References

- Source: `Views/*.fs`
- Avalonia custom rendering: https://docs.avaloniaui.net/docs/next/concepts/custom-rendering/
- SkiaSharp in Avalonia: use `ISkiaDrawingContextImpl` from `Avalonia.Skia`
- ReactiveUI `ReactiveUserControl<T>`: https://www.reactiveui.net/docs/handbook/usercontrols/
