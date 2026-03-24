# 03 — Migrate Type Definitions (`def.fs`)

**Status:** pending

## Overview

`def.fs` is the largest file (866 lines) and the most F#-idiomatic. It defines nearly all core data types: discriminated unions (DUs), struct records, active patterns for parsing, and the `RedrawCommand` DU with 30+ cases. This task translates them all to C#.

## Motivation

F# discriminated unions have no direct C# equivalent. The idiomatic C# approach for a DU with many cases is sealed class hierarchies with `abstract` base + `record` subclasses, matched with `switch` expressions. The `[<Struct>]` DUs (stack-allocated, no heap allocation) must be translated to C# `readonly struct` or `record struct` types. The 30+ case `RedrawCommand` DU is the most complex — it drives the entire redraw pipeline and is matched exhaustively in `model.fs`.

## Steps

### Struct Records (F# `[<Struct>]` + record)

1. **`Request`** → `readonly record struct`:
   ```csharp
   public readonly record struct Request(string Method, object[] Parameters);
   ```

2. **`ModeInfo`** → `readonly record struct`:
   ```csharp
   public readonly record struct ModeInfo(
       CursorShape? CursorShape, int? CellPercentage,
       int? BlinkWait, int? BlinkOn, int? BlinkOff,
       int? AttrId, int? AttrIdLm,
       string ShortName, string Name);
   ```

3. **`RgbAttr`** → `readonly record struct` with a static `Empty`:
   ```csharp
   public readonly record struct RgbAttr(
       Color? Foreground, Color? Background, Color? Special,
       bool Reverse, bool Italic, bool Bold, bool Underline, bool Undercurl)
   {
       public static readonly RgbAttr Empty = new(null, null, null, false, false, false, false, false);
   }
   ```

4. **`GridCell`** → `readonly record struct` (note `int voption` → `int?` / `ValueOption` helper):
   ```csharp
   public readonly record struct GridCell(Rune Text, int? HlId, int? Repeat);
   ```

5. **`GridLine`** → `readonly record struct`:
   ```csharp
   public readonly record struct GridLine(int Grid, int Row, int ColStart, GridCell[] Cells, bool Wrap);
   ```

6. **`HighlightAttr`** → `record struct`:
   ```csharp
   public record struct HighlightAttr(int Id, RgbAttr RgbAttr, RgbAttr CtermAttr, object[] Info)
   {
       public static readonly HighlightAttr Default = new(-1, RgbAttr.Empty, RgbAttr.Empty, Array.Empty<object>());
   }
   ```

7. **`Extmark`** → `record struct`:
   ```csharp
   public readonly record struct Extmark(int Ns, int Mark, int Row, int Col);
   ```

8. **`CompleteItem`** → `record struct`:
   ```csharp
   public readonly record struct CompleteItem(string Word, string Abbr, string Menu, string Info)
   {
       public static readonly CompleteItem Empty = new("", "", "", "");
       public int GetLength() => Word.Length + Abbr.Length + Menu.Length + Info.Length;
   }
   ```

### Struct Discriminated Unions (simple enumerations)

9. **`CursorShape`** → `enum`:
   ```csharp
   public enum CursorShape { Block, Horizontal, Vertical }
   ```

10. **`AmbiWidth`** → `enum`:
    ```csharp
    public enum AmbiWidth { Single, Double }
    ```

11. **`ShowTabline`** → `enum`:
    ```csharp
    public enum ShowTabline { Never, AtLeastTwo, Always }
    ```

12. **`Anchor`** → `enum`:
    ```csharp
    public enum Anchor { NorthWest, NorthEast, SouthWest, SouthEast }
    ```

13. **`VimCompleteKind`** → `enum`:
    ```csharp
    public enum VimCompleteKind { Variable, Function, Member, Typedef, Macro }
    ```

14. **`SemanticHighlightGroup`** → `enum` (already enum-like in F#, keep integer values as-is):
    ```csharp
    public enum SemanticHighlightGroup { SpecialKey = 0, EndOfBuffer = 1, /* ... */ NormalFloat = 48 }
    ```

15. **`SignKind`** → `enum` (already enum in F#):
    ```csharp
    public enum SignKind { Add = 0, Change = 1, Delete = 2, Warning = 3, Error = 4, Other = 5 }
    ```

### Complex Discriminated Unions

16. **`Event` DU** → sealed hierarchy:
    ```csharp
    public abstract record Event;
    public sealed record RpcRequestEvent(int ReqId, Request Req, Func<int, Result<object,object>, Task> Handler) : Event;
    public sealed record RpcResponseEvent(int RspId, Result<object,object> Rsp) : Event;
    public sealed record NotificationEvent(Request Req) : Event;
    public sealed record StdErrorEvent(string Message) : Event;
    public sealed record CrashEvent(int Code) : Event;
    public sealed record ByteMessageEvent(byte Message) : Event;
    public sealed record UnhandledExceptionEvent(Exception Ex) : Event;
    public sealed record ExitEvent(int Code) : Event;
    ```

17. **`UiOption` DU** → sealed hierarchy:
    ```csharp
    public abstract record UiOption;
    public sealed record ArabicShapeOption(bool Value) : UiOption;
    public sealed record AmbiWidthOption(AmbiWidth Value) : UiOption;
    public sealed record EmojiOption(bool Value) : UiOption;
    public sealed record GuifontOption(string Value) : UiOption;
    public sealed record GuifontSetOption(List<string> Value) : UiOption;
    public sealed record GuifontWideOption(string Value) : UiOption;
    public sealed record LineSpaceOption(int Value) : UiOption;
    public sealed record ShowTablineOption(ShowTabline Value) : UiOption;
    public sealed record TermGuiColorsOption(bool Value) : UiOption;
    public sealed record UnknownOptionValue(object Value) : UiOption;
    ```

18. **`Rune` DU** → sealed hierarchy with custom `Equals`/`GetHashCode`:
    The F# `Rune` has `[<CustomEquality>]` and methods `Codepoint`, `feed(buf, ref len)`. Translate as:
    ```csharp
    public abstract class Rune : IEquatable<Rune> {
        public static readonly Rune Empty = new SingleChar(' ');
        public abstract uint Codepoint { get; }
        public abstract void Feed(char[] buf, ref int len);
        public abstract void Feed(uint[] buf, ref int len);
        public abstract bool Equals(Rune? other);
        public override bool Equals(object? obj) => obj is Rune r && Equals(r);
    }
    public sealed class SingleChar : Rune {
        public char Chr { get; }
        public SingleChar(char chr) { Chr = chr; }
        public override uint Codepoint => (uint)Chr;
        public override string ToString() => Chr.ToString();
        public override void Feed(char[] buf, ref int len) { buf[len++] = Chr; }
        public override void Feed(uint[] buf, ref int len) { buf[len++] = Codepoint; }
        public override bool Equals(Rune? other) => other is SingleChar s && Chr == s.Chr;
        public override int GetHashCode() => Chr.GetHashCode();
    }
    public sealed class SurrogatePair : Rune { ... }
    public sealed class Composed : Rune { ... }
    ```

19. **`RedrawCommand` DU (30+ cases)** → sealed hierarchy. This is the largest migration item:
    ```csharp
    public abstract record RedrawCommand;
    public sealed record SetOptionCmd(UiOption[] Options) : RedrawCommand;
    public sealed record SetTitleCmd(string Title) : RedrawCommand;
    public sealed record SetIconCmd(string Icon) : RedrawCommand;
    public sealed record ModeInfoSetCmd(bool CursorStyleEnabled, ModeInfo[] ModeInfo) : RedrawCommand;
    public sealed record ModeChangeCmd(string Name, int Index) : RedrawCommand;
    public sealed record MouseCmd(bool On) : RedrawCommand;
    public sealed record BusyCmd(bool On) : RedrawCommand;
    public sealed record BellCmd : RedrawCommand;
    public sealed record VisualBellCmd : RedrawCommand;
    public sealed record FlushCmd : RedrawCommand;
    public sealed record DefaultColorsSetCmd(Color Fg, Color Bg, Color Sp, Color CtermFg, Color CtermBg) : RedrawCommand;
    public sealed record HighlightAttrDefineCmd(HighlightAttr[] Attrs) : RedrawCommand;
    public sealed record GridLineCmd(ReadOnlyMemory<GridLine> Lines) : RedrawCommand;
    public sealed record GridClearCmd(int Grid) : RedrawCommand;
    public sealed record GridDestroyCmd(int Grid) : RedrawCommand;
    public sealed record GridCursorGotoCmd(int Grid, int Row, int Col) : RedrawCommand;
    public sealed record GridScrollCmd(int Grid, int Top, int Bot, int Left, int Right, int Rows, int Cols) : RedrawCommand;
    public sealed record GridResizeCmd(int Grid, int Width, int Height) : RedrawCommand;
    public sealed record WinPosCmd(int Grid, int Win, int StartRow, int StartCol, int Width, int Height) : RedrawCommand;
    public sealed record WinFloatPosCmd(int Grid, int Win, Anchor Anchor, int AnchorGrid, double AnchorRow, double AnchorCol, bool Focusable, int ZIndex) : RedrawCommand;
    public sealed record WinExternalPosCmd(int Grid, int Win) : RedrawCommand;
    public sealed record WinHideCmd(int Grid) : RedrawCommand;
    public sealed record WinScrollOverStartCmd : RedrawCommand;
    public sealed record WinScrollOverResetCmd : RedrawCommand;
    public sealed record WinCloseCmd(int Grid) : RedrawCommand;
    public sealed record WinViewportCmd(int Grid, int Win, int TopLine, int BotLine, int CurLine, int CurCol, int LineCount) : RedrawCommand;
    public sealed record WinExtmarksCmd(int Grid, int Win, Extmark[] Marks) : RedrawCommand;
    public sealed record MsgSetPosCmd(int Grid, int Row, bool Scrolled, string SepChar) : RedrawCommand;
    public sealed record PopupMenuShowCmd(CompleteItem[] Items, int Selected, int Row, int Col, int Grid) : RedrawCommand;
    public sealed record PopupMenuSelectCmd(int Selected) : RedrawCommand;
    public sealed record PopupMenuHideCmd : RedrawCommand;
    public sealed record SemanticHighlightGroupSetCmd(Dictionary<SemanticHighlightGroup, int> Groups) : RedrawCommand;
    public sealed record UnknownCommandCmd(object Data) : RedrawCommand;
    public sealed record MultiRedrawCommandCmd(RedrawCommand[] Commands) : RedrawCommand;
    ```

20. **`Response` type alias**: `type Response = Result<obj, obj>` → define a `Result<T,E>` struct or use the pattern directly. Since F#'s `Result<T,E>` is often used, create a minimal C# equivalent:
    ```csharp
    public readonly struct Result<T, E> {
        public bool IsOk { get; }
        public T Value { get; }
        public E Error { get; }
        public static Result<T,E> Ok(T v) => new(true, v, default!);
        public static Result<T,E> Err(E e) => new(false, default!, e);
    }
    public static class Result {
        public static Result<T,E> Ok<T,E>(T v) => Result<T,E>.Ok(v);
        public static Result<T,E> Err<T,E>(E e) => Result<T,E>.Err(e);
    }
    ```

21. **`EventParseException`**:
    ```csharp
    public class EventParseException(object data) : Exception($"Could not parse the neovim message: {data}")
    {
        public object Input { get; } = data;
    }
    ```

22. **Active patterns for parsing** (`C`, `C1`, `P`, `PX`, `KV`, `FindKV`, `HighlightAttr`, etc.) → static helper methods in a `Parsers` static class:
    ```csharp
    public static class Parsers {
        // (|C|_|): match ObjArray [| string cmd; rest... |]
        public static bool TryMatchC(object x, out string cmd, out object[] rest) { ... }
        // (|KV|_|) k: match ObjArray [| string k; value |]
        public static bool TryMatchKV(string k, object x, out object value) { ... }
        // FindKV
        public static object? FindKV(string k, object x) { ... }
        // parse_uioption
        public static UiOption? ParseUiOption(object x) { ... }
        // parse_default_colors
        public static RedrawCommand? ParseDefaultColors(object x) { ... }
        // parse_mode_info
        public static ModeInfo? ParseModeInfo(object x) { ... }
        // parse_hi_attr
        public static HighlightAttr? ParseHiAttr(object x) { ... }
        // (|Rune|_|) - the msgpack rune parser
        public static bool TryParseRune(object x, out Rune? rune) { ... }
        // parse_grid_line
        public static GridLine? ParseGridLine(object x) { ... }
    }
    ```

23. **`LineHeightOption` DU** (used in `states.fs`):
    ```csharp
    public abstract record LineHeightOption {
        public static readonly LineHeightOption Default = new DefaultHeight();
    }
    public sealed record DefaultHeight : LineHeightOption;
    public sealed record AbsoluteHeight(double Value) : LineHeightOption;
    public sealed record RelativeHeight(double Value) : LineHeightOption;
    ```

24. **`BackgroundComposition` DU** (used in `states.fs`/`def.fs`):
    ```csharp
    public enum BackgroundComposition { NoComposition, Blur, Acrylic, Transparent }
    ```

## References

- Source: `def.fs` (lines 1–866)
- C# records: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record
- Switch expressions: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/switch-expression
