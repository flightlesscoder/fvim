# 07 — Migrate Global State (`states.fs`)

**Status:** pending

## Overview

Translate `states.fs` to `States.cs`. The F# module defines ~30 mutable top-level variables that serve as global application state for keyboard config, clipboard, cursor animation, font rendering, UI feature flags, and background composition settings.

## Motivation

F# modules with `let mutable` top-level bindings map directly to `static` fields or properties in a C# `static class`. F# `[<Literal>]` bindings map to `const`. The `Set.empty<string>` type maps to `HashSet<string>`. The `PopulateUIOptions` function is a straightforward translation.

## Steps

1. **Create `States.cs`** as `public static class States`.

2. **Translate `[<Literal>]` bindings** to `const string`:
   ```csharp
   public const string UiOptRgb           = "rgb";
   public const string UiOptExtLinegrid   = "ext_linegrid";
   public const string UiOptExtMultigrid  = "ext_multigrid";
   public const string UiOptExtPopupmenu  = "ext_popupmenu";
   public const string UiOptExtTabline    = "ext_tabline";
   public const string UiOptExtCmdline    = "ext_cmdline";
   public const string UiOptExtWildmenu   = "ext_wildmenu";
   public const string UiOptExtMessages   = "ext_messages";
   public const string UiOptExtHlstate    = "ext_hlstate";
   public const string UiOptExtTermcolors = "ext_termcolors";
   public const string UiOptExtWindows    = "ext_windows";
   ```

3. **Translate `let mutable` bindings** to `static` fields/properties:
   ```csharp
   // channel
   public static int ChannelId = 1;

   // keyboard mapping
   public static bool KeyDisableShiftSpace = false;
   public static bool KeyAutoIme = false;
   public static bool KeyAltGr = false;

   // clipboard
   public static string[] ClipboardLines = Array.Empty<string>();
   public static string ClipboardRegtype = "";

   // cursor
   public static bool CursorSmoothMove = false;
   public static bool CursorSmoothBlink = false;

   // font rendering
   public static bool FontAntialias     = true;
   public static bool FontDrawBounds    = false;
   public static bool FontAutohint      = false;
   public static bool FontSubpixel      = true;
   public static bool FontAutosnap      = true;
   public static bool FontLigature      = true;
   public static SKPaintHinting FontHintLevel  = SKPaintHinting.NoHinting;
   public static FontWeight FontWeightNormal   = FontWeight.Normal;
   public static FontWeight FontWeightBold     = FontWeight.Bold;
   public static LineHeightOption FontLineheight = LineHeightOption.Default;
   public static bool FontNonerd        = false;

   // ui feature flags
   public static HashSet<string> UiAvailableOpts = new();
   public static bool UiMultigrid   = true;
   public static bool UiPopupmenu   = true;
   public static bool UiTabline     = false;
   public static bool UiCmdline     = false;
   public static bool UiWildmenu    = false;
   public static bool UiMessages    = false;
   public static bool UiTermcolors  = false;
   public static bool UiHlstate     = false;

   // background
   public static BackgroundComposition BackgroundComposition = BackgroundComposition.NoComposition;
   public static double BackgroundOpacity    = 1.0;
   public static double BackgroundAltOpacity = 1.0;
   public static string BackgroundImageFile  = "";
   public static double BackgroundImageOpacity = 1.0;
   public static Stretch BackgroundImageStretch = Stretch.None;
   public static HorizontalAlignment BackgroundImageHalign = HorizontalAlignment.Left;
   public static VerticalAlignment BackgroundImageValign   = VerticalAlignment.Top;

   // defaults
   public static int DefaultWidth  = 800;
   public static int DefaultHeight = 600;
   ```

4. **Translate `Set.empty<string>`** → `HashSet<string>`:
   The F# `ui_available_opts` is `Set.empty<string>` (immutable). It is populated at startup via `Set.add`. In C# use a `HashSet<string>` for O(1) `Contains` and mutability:
   ```csharp
   public static HashSet<string> UiAvailableOpts = new(StringComparer.Ordinal);
   ```
   Replace `Set.add k s` with `UiAvailableOpts.Add(k)`.
   Replace `Set.contains k s` with `UiAvailableOpts.Contains(k)`.

5. **Translate `PopulateUIOptions`**:
   ```csharp
   public static void PopulateUiOptions(Dictionary<string, object> opts) {
       void C(string k, object v) {
           if (UiAvailableOpts.Contains(k)) opts[k] = v;
       }
       C(UiOptExtPopupmenu,  UiPopupmenu);
       C(UiOptExtMultigrid,  UiMultigrid);
       C(UiOptExtTabline,    UiTabline);
       C(UiOptExtCmdline,    UiCmdline);
       C(UiOptExtWildmenu,   UiWildmenu);
       C(UiOptExtMessages,   UiMessages);
       C(UiOptExtHlstate,    UiHlstate);
       C(UiOptExtTermcolors, UiTermcolors);
   }
   ```

6. **Delete the `type Foo = A`** at the bottom of `states.fs` — it is a stub type used to work around an F# compiler quirk and is not needed in C#.

7. **Update all usages** across the codebase: Replace `states.channel_id` with `States.ChannelId`, etc. Use a case-by-case find/replace rather than regex to avoid false positives.

## References

- Source: `states.fs`
- F# `Set` → C# `HashSet`: O(log n) vs O(1), but semantics differ (Set is immutable). Use `HashSet` for performance.
