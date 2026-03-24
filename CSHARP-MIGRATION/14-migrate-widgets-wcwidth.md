# 14 — Migrate Widgets and Unicode Width (`widgets.fs`, `wcwidth.fs`)

**Status:** pending

## Overview

Translate `widgets.fs` (174 lines) and `wcwidth.fs` (604 lines) to C#. `wcwidth.fs` is a Unicode character width lookup table — a large data array with a binary search algorithm. `widgets.fs` defines the `WidgetPlacement` record and layout helpers for UI widgets placed on the grid.

## Motivation

Both files are relatively mechanical translations. `wcwidth.fs` contains two large constant arrays (`ZeroWidth[]` and `WidthRange[]`) of Unicode ranges. These translate directly to C# static readonly arrays. `widgets.fs` uses a record with member functions and pattern matching on draw bounds.

## Steps

### wcwidth.fs

1. **Create `WcWidth.cs`** as `public static class WcWidth`.

2. **Translate the `ZeroWidth` array** — the F# version is an array of `(lo, hi)` int pairs for zero-width character ranges. In C#:
   ```csharp
   private static readonly (int Lo, int Hi)[] ZeroWidth = {
       (0x0300, 0x036F), (0x0483, 0x0486), // ...
       // copy all ranges from wcwidth.fs
   };
   ```

3. **Translate the `WidthRange` array** — similar to ZeroWidth but for full-width (double-width) characters:
   ```csharp
   private static readonly (int Lo, int Hi)[] FullWidth = {
       (0x1100, 0x115F), (0x2329, 0x232A), // ...
       // copy all ranges from wcwidth.fs
   };
   ```

4. **Translate `GetWidthAttr(codepoint)`** — binary search on ranges:
   ```csharp
   public static int GetWidth(uint codepoint) {
       if (codepoint == 0) return 0;
       // check ZeroWidth
       if (BinarySearch(ZeroWidth, codepoint)) return 0;
       // check FullWidth
       if (BinarySearch(FullWidth, codepoint)) return 2;
       return 1;
   }

   private static bool BinarySearch((int Lo, int Hi)[] ranges, uint cp) {
       int lo = 0, hi = ranges.Length - 1;
       while (lo <= hi) {
           int mid = (lo + hi) / 2;
           if (cp < (uint)ranges[mid].Lo) hi = mid - 1;
           else if (cp > (uint)ranges[mid].Hi) lo = mid + 1;
           else return true;
       }
       return false;
   }
   ```

5. **Translate `GetWidthAttr(rune)`** — the Rune-level width function that delegates to the codepoint version and handles `Composed` runes:
   ```csharp
   public static int GetWidthAttr(Rune rune) => rune switch {
       SingleChar { Chr: var c }         => GetWidth((uint)c),
       SurrogatePair { } sp              => GetWidth(sp.Codepoint),
       Composed { Runes: var xs }        => xs.Sum(r => GetWidthAttr(r)),
       _                                 => 1
   };
   ```

### widgets.fs

6. **Create `Widgets.cs`**.

7. **Translate `WidgetPlacement` record**:
   ```csharp
   public record WidgetPlacement(
       string Svg,
       string Text,
       int HlGroup,
       int Row,
       int Col,
       int Width,
       int Height,
       bool Sticky  // whether to anchor to top or bottom of cell
   ) {
       public HorizontalAlignment HorizontalAlignment => ...;
       public VerticalAlignment VerticalAlignment => ...;
       public bool Stretch => ...;
       public Rect GetDrawingBounds(double glyphW, double glyphH) => ...;
       public (Color fg, Color bg) GetTextAttr() => ...;
       public (Color fg, Color bg) GetSvgAttr() => ...;
   }
   ```

8. **Translate the `GetDrawingBounds` logic**: This computes pixel bounds for a widget given glyph dimensions. Read `widgets.fs` carefully for the exact formula.

9. **Translate `GetTextAttr` / `GetSvgAttr`**: These delegate to `theme.GetDrawAttrs` to get colors for the widget's highlight group.

## References

- Source: `wcwidth.fs`, `widgets.fs`
- Unicode East Asian Width: https://www.unicode.org/reports/tr11/
- The `wcwidth.fs` data is derived from `wcwidth.c` by Markus Kuhn — the data arrays must be copied verbatim (same character ranges)
