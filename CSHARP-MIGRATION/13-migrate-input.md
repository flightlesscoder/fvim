# 13 — Migrate Input Handling (`input.fs`)

**Status:** pending

## Overview

Translate `input.fs` to `Input.cs`. This is a 475-line module that maps Avalonia keyboard and mouse events to Neovim's input string notation (e.g. `<C-S-F2>`, `<LeftMouse>`, `<ScrollWheelUp>`). It uses active patterns extensively for modifier key checking, and large pattern-match expressions for key-to-Vim-notation mapping.

## Motivation

Active patterns (`(|HasFlag|_|)`, `(|NoFlag|_|)`, `(|NvimSupportedMouseButton|_|)`, `(|DoesntBlockTextInput|_|)`) become inline `if` conditions or extension methods. The key mapping tables are long match expressions that become `switch` expressions or `Dictionary` lookups. The `KB()`, `MB()`, `DIR()` helper functions become static methods.

## Steps

1. **Create `Input.cs`** as `public static class Input`.

2. **Translate modifier active patterns**:
   ```csharp
   // (|HasFlag|_|) flag x → x.HasFlag(flag)
   // (|NoFlag|_|) flag x → !x.HasFlag(flag)
   // (|DoesntBlockTextInput|_|) x
   public static bool DoesntBlockTextInput(KeyModifiers x) {
       if (States.KeyAltGr && x.HasFlag(KeyModifiers.Alt) && x.HasFlag(KeyModifiers.Control))
           return true;
       return !x.HasFlag(KeyModifiers.Alt) && !x.HasFlag(KeyModifiers.Control) && !x.HasFlag(KeyModifiers.Meta);
   }
   ```

3. **Translate `(|NvimSupportedMouseButton|_|)`**:
   ```csharp
   public static bool IsNvimSupportedMouseButton(MouseButton mb, out string name) {
       switch (mb) {
           case MouseButton.Left:   name = "left";   return true;
           case MouseButton.Right:  name = "right";  return true;
           case MouseButton.Middle: name = "middle"; return true;
           default: name = ""; return false;
       }
   }
   ```

4. **Translate the key mapping table** — this is the large `match key with` expression that maps `Avalonia.Input.Key` to Neovim key names. Convert to a static `Dictionary<Key, string>` initialized once:
   ```csharp
   private static readonly Dictionary<Key, string> _keyMap = new() {
       [Key.F1]  = "F1",  [Key.F2]  = "F2",  /* ... */ [Key.F12] = "F12",
       [Key.Up]       = "Up",
       [Key.Down]     = "Down",
       [Key.Left]     = "Left",
       [Key.Right]    = "Right",
       [Key.Back]     = "BS",
       [Key.Tab]      = "Tab",
       [Key.Return]   = "CR",
       [Key.Escape]   = "Esc",
       [Key.Delete]   = "Del",
       [Key.Home]     = "Home",
       [Key.End]      = "End",
       [Key.PageUp]   = "PageUp",
       [Key.PageDown] = "PageDown",
       [Key.Insert]   = "Insert",
       [Key.Space]    = "Space",
       // ... all keys from the F# match table
   };
   ```

5. **Translate `KB()` (key binding formatter)**:
   ```csharp
   public static string? KB(KeyEventArgs e) {
       var key = e.Key;
       var mods = e.KeyModifiers;
       // special characters that need <lt>, <Bar>, etc.
       if (key == Key.OemComma && mods == KeyModifiers.None) return "<lt>";
       if (!_keyMap.TryGetValue(key, out var name)) return null;
       var prefix = BuildModifierPrefix(mods);
       if (string.IsNullOrEmpty(prefix) && name.Length == 1) return name;
       return $"<{prefix}{name}>";
   }

   private static string BuildModifierPrefix(KeyModifiers mods) {
       var sb = new StringBuilder();
       if (mods.HasFlag(KeyModifiers.Shift))   sb.Append('S').Append('-');
       if (mods.HasFlag(KeyModifiers.Control)) sb.Append('C').Append('-');
       if (mods.HasFlag(KeyModifiers.Alt))     sb.Append('M').Append('-');
       if (mods.HasFlag(KeyModifiers.Meta))    sb.Append('D').Append('-');
       return sb.ToString();
   }
   ```

6. **Translate `MB()` (mouse button formatter)**:
   ```csharp
   public static string MB(string buttonName, KeyModifiers mods, int count) {
       var prefix = BuildModifierPrefix(mods);
       return count > 1
           ? $"<{prefix}{count}-{buttonName}Mouse>"
           : $"<{prefix}{buttonName}Mouse>";
   }
   ```

7. **Translate `DIR()` (scroll direction formatter)**:
   ```csharp
   public static string DIR(PointerWheelEventArgs e, KeyModifiers mods) {
       var prefix = BuildModifierPrefix(mods);
       return e.Delta.Y > 0 ? $"<{prefix}ScrollWheelUp>" : $"<{prefix}ScrollWheelDown>";
   }
   ```

8. **Translate `InputEvent` DU**:
   ```csharp
   public abstract record InputEvent;
   public sealed record KeyInputEvent(string Key) : InputEvent;
   public sealed record MousePressEvent(string Button) : InputEvent;
   public sealed record MouseReleaseEvent(string Button) : InputEvent;
   public sealed record MouseDragEvent(string Button) : InputEvent;
   public sealed record MouseWheelEvent(string Direction) : InputEvent;
   public sealed record TextInputEvent(string Text) : InputEvent;
   ```

9. **Handle `#nowarn "0058"`**: This suppresses F# incomplete match warnings. In C#, ensure all switch expressions have a default arm or use discard patterns.

10. **Translate `#nowarn "0025"`** usages in `GridViewModel.fs` — same warning, not applicable in C# switch.

11. **Handle the AltGr key combination**: The F# code has special handling for `key_altGr` + `Alt+Ctrl` to allow AltGr to pass through as text input on European keyboards. Preserve this logic exactly.

## References

- Source: `input.fs`
- Avalonia input docs: https://docs.avaloniaui.net/docs/input/keyboard
- Neovim key notation: https://neovim.io/doc/user/intro.html#key-notation
