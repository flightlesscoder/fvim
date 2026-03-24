# 12 — Migrate UI Core, Theme, and SVG (`ui.fs`, `theme.fs`, `svg.fs`)

**Status:** pending

## Overview

Translate three related modules: `ui.fs` (font management and core UI abstractions), `theme.fs` (highlight/color system), and `svg.fs` (SVG icon rendering). These modules define interfaces (`IGridUI`, `IFrame`), event streams for theme/font changes, and the global highlight array.

## Motivation

These modules are heavily interface/event-driven. F# `Event<'T>()` becomes `Subject<T>` from `System.Reactive`. F# interfaces translate directly to C# interfaces. The `theme.fs` module has global mutable arrays (`hi_defs[]`, `mode_defs[]`) and a map — translate to `static` arrays and a `Dictionary`.

## Steps

### ui.fs

1. **Translate `IGridUI` interface** (from `def.fs` or `ui.fs`):
   ```csharp
   public interface IGridUI {
       int Id { get; }
       int GridHeight { get; }
       int GridWidth { get; }
       IObservable<IGridUI> Resized { get; }
       IObservable<(int gridId, InputEvent input, RoutedEventArgs args)> Input { get; }
       Color BackgroundColor { get; }
       bool HasChildren { get; }
       double RenderScale { get; }
       void Redraw(RedrawCommand command);
       IGridUI CreateChild(int id, int rows, int cols);
       void AddChild(IGridUI child);
       void RemoveChild(IGridUI child);
       void Detach();
   }
   ```

2. **Translate `IFrame` interface**:
   ```csharp
   public interface IFrame {
       string Title { get; set; }
       IGridUI MainGrid { get; }
       void Sync(RedrawCommand cmd);
   }
   ```

3. **Translate `WindowLayout` DU**:
   ```csharp
   public abstract record WindowLayout;
   public sealed record WindowRecord(IGridUI Grid) : WindowLayout;
   public sealed record VSplitLayout(List<WindowLayout> Children) : WindowLayout;
   public sealed record HSplitLayout(List<WindowLayout> Children) : WindowLayout;
   ```

4. **Translate font management** — `DefaultFont`, `DefaultFontWide`, `DefaultFontEmoji`:
   ```csharp
   public static class UI {
       public static Typeface DefaultFont { get; private set; } = Typeface.Default;
       public static Typeface DefaultFontWide { get; private set; } = Typeface.Default;
       public static Typeface DefaultFontEmoji { get; private set; } = Typeface.Default;
       // Font cache
       private static readonly Dictionary<(string,FontStyle,FontWeight), Typeface> _typefaceCache = new();
       public static Typeface GetTypeface(string family, FontStyle style, FontWeight weight) { ... }
   }
   ```

5. **Translate `GridBufferCell` and `GridSize`/`GridRect`**:
   ```csharp
   public struct GridBufferCell {
       public Rune Text;
       public int HlId;
       public List<Extmark> Marks;
       public static GridBufferCell[,] CreateGrid(int rows, int cols) {
           var grid = new GridBufferCell[rows, cols];
           // initialize with space character and default hlid
           return grid;
       }
   }
   public readonly record struct GridSize(int Rows, int Cols);
   public readonly record struct GridRect(int Row, int Col, int Height, int Width) {
       public bool Contains(int r, int c) => r >= Row && r < Row+Height && c >= Col && c < Col+Width;
       public bool Disjoint(GridRect other) => ... ;
   }
   ```

### theme.fs

6. **Translate global highlight state**:
   ```csharp
   public static class Theme {
       public static HighlightAttr[] HiDefs = new HighlightAttr[16384]; // or appropriate size
       public static ModeInfo[] ModeDefs = Array.Empty<ModeInfo>();
       public static Dictionary<SemanticHighlightGroup, int> Semhl = new();

       private static readonly Subject<Unit> _hlChangeSubject = new();
       private static readonly Subject<Unit> _fontConfigSubject = new();
       private static readonly Subject<bool> _cursorEnSubject = new();
       private static readonly Subject<Unit> _themeConfigSubject = new();

       public static IObservable<Unit> HlChanged => _hlChangeSubject;
       public static IObservable<Unit> FontConfigChanged => _fontConfigSubject;
       public static IObservable<bool> CursorEnabled => _cursorEnSubject;
       public static IObservable<Unit> ThemeConfigChanged => _themeConfigSubject;

       public static double FontSize { get; private set; } = 11.0;
   }
   ```

7. **Translate `GetDrawAttrs(hlid, reverse)`**:
   ```csharp
   public static (Color fg, Color bg, Color sp, bool bold, bool italic, bool underline, bool undercurl)
       GetDrawAttrs(int hlId, bool reverse) {
       var attr = hlId >= 0 && hlId < HiDefs.Length ? HiDefs[hlId] : HighlightAttr.Default;
       var rgb = attr.RgbAttr;
       // apply reverse
       var fg = reverse ? (rgb.Background ?? DefaultBg) : (rgb.Foreground ?? DefaultFg);
       var bg = reverse ? (rgb.Foreground ?? DefaultFg) : (rgb.Background ?? DefaultBg);
       return (fg, bg, rgb.Special ?? DefaultSp, rgb.Bold, rgb.Italic, rgb.Underline, rgb.Undercurl);
   }
   ```

8. **Translate `getSemanticHighlightGroup()`**:
   ```csharp
   public static int GetSemanticHighlightGroup(SemanticHighlightGroup group) =>
       Semhl.TryGetValue(group, out var id) ? id : 0;
   ```

### svg.fs

9. **Translate SVG rendering**: `svg.fs` loads SVG icons for sign gutter marks. It uses `Avalonia.Svg.Skia.SvgImage`. Translate the loading and caching logic:
   ```csharp
   public static class Svg {
       private static readonly Dictionary<string, IImage> _cache = new();
       public static IImage? LoadSvg(string path) {
           if (_cache.TryGetValue(path, out var img)) return img;
           // use Avalonia.Svg.Skia to load
           ...
       }
   }
   ```

## References

- Source: `ui.fs`, `theme.fs`, `svg.fs`
- `System.Reactive.Subjects.Subject<T>`: replaces F# `Event<T>`
- Avalonia.Svg package: unchanged
