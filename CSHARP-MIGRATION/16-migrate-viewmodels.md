# 16 — Migrate ViewModels

**Status:** pending

## Overview

Translate all ViewModels from F# to C#. The ViewModel hierarchy is:
- `ViewModelBase.fs` → `ViewModelBase.cs`
- `ThemableViewModelBase.fs` → `ThemableViewModelBase.cs`
- `CursorViewModel.fs` → `CursorViewModel.cs`
- `CompletionItemViewModel.fs` → `CompletionItemViewModel.cs`
- `PopupMenuViewModel.fs` → `PopupMenuViewModel.cs`
- `GridViewModel.fs` → `GridViewModel.cs` (most complex, ~700+ lines)
- `FrameViewModel.fs` → `FrameViewModel.cs`
- `TitleBarViewModel.fs` → `TitleBarViewModel.cs`
- `CrashReportViewModel.fs` → `CrashReportViewModel.cs`

## Motivation

F# ViewModels use ReactiveUI's `ReactiveObject` base class, which is identical in C#. `let mutable` fields become private fields with `RaiseAndSetIfChanged`. F# uses `this.RaiseAndSetIfChanged` as a method call; in C# this is typically done via `SetAndRaise` or `RaisePropertyChanged`. The `#nowarn "0025"` incomplete match warnings become switch default arms.

## Steps

### ViewModelBase.fs

1. **Create `ViewModelBase.cs`**:
   ```csharp
   public class ViewModelBase : ReactiveObject {
       private double _x, _y, _width, _height;
       private int _renderTick;

       public double X { get => _x; set => this.RaiseAndSetIfChanged(ref _x, value); }
       public double Y { get => _y; set => this.RaiseAndSetIfChanged(ref _y, value); }
       public double Width  { get => _width;  set => this.RaiseAndSetIfChanged(ref _width, value); }
       public double Height { get => _height; set => this.RaiseAndSetIfChanged(ref _height, value); }
       public int RenderTick { get => _renderTick; set => this.RaiseAndSetIfChanged(ref _renderTick, value); }
   }
   ```

### ThemableViewModelBase.fs

2. **Create `ThemableViewModelBase.cs`** — inherits `ViewModelBase`, subscribes to `Theme.HlChanged` and `Theme.ThemeConfigChanged` to update color brushes:
   ```csharp
   public class ThemableViewModelBase : ViewModelBase {
       private IBrush _normalFg = Brushes.White;
       private IBrush _normalBg = Brushes.Black;
       // ... SelectFg, SelectBg, HoverBg, ScrollbarFg, ScrollbarBg, BorderBrush, InactiveFg

       public IBrush NormalFg { get => _normalFg; private set => this.RaiseAndSetIfChanged(ref _normalFg, value); }
       // ...

       protected ThemableViewModelBase() {
           Theme.HlChanged.Subscribe(_ => UpdateColors());
           Theme.ThemeConfigChanged.Subscribe(_ => UpdateColors());
           UpdateColors();
       }

       protected virtual void UpdateColors() {
           var (fg, bg, _, _, _, _, _) = Theme.GetDrawAttrs(Theme.GetSemanticHighlightGroup(SemanticHighlightGroup.Pmenu), false);
           NormalFg = new SolidColorBrush(fg);
           NormalBg = new SolidColorBrush(bg);
           // ... update all brushes
       }
   }
   ```

### CursorViewModel.fs

3. **Translate `CursorViewModel`**: Manages cursor position, shape (block/horizontal/vertical), blink state, and color. Uses `DispatcherTimer` for blink animation.
   ```csharp
   public class CursorViewModel : ViewModelBase {
       private CursorShape _shape = CursorShape.Block;
       private int _cellPercentage = 100;
       private bool _isVisible = true;
       private DispatcherTimer? _blinkTimer;
       // ...
   }
   ```

### CompletionItemViewModel.fs

4. **Translate `CompletionItemViewModel`**: Simple ViewModel wrapping `CompleteItem`:
   ```csharp
   public class CompletionItemViewModel(CompleteItem item) : ViewModelBase {
       public string Word  => item.Word;
       public string Abbr  => item.Abbr;
       public string Menu  => item.Menu;
       public string Info  => item.Info;
       public VimCompleteKind Kind => ParseKind(item.Abbr);
   }
   ```

### PopupMenuViewModel.fs

5. **Translate `PopupMenuViewModel`**: Manages the completion popup list state:
   ```csharp
   public class PopupMenuViewModel : ThemableViewModelBase {
       private bool _isVisible = false;
       private int _selectedIndex = -1;
       public ObservableCollection<CompletionItemViewModel> Items { get; } = new();
       // ...
   }
   ```

### GridViewModel.fs

6. **Translate `GridViewModel`** — the most complex ViewModel (~700 lines of F# across the file and its methods). This implements `IGridUI`.

   **Key fields**:
   ```csharp
   public class GridViewModel : ViewModelBase, IGridUI {
       private readonly int _gridId;
       private readonly CursorViewModel _cursorVm;
       private readonly PopupMenuViewModel _popupMenuVm;
       private readonly List<GridViewModel> _childGrids = new();
       private readonly Subject<IGridUI> _resizeSubject = new();
       private readonly Subject<(int, InputEvent, RoutedEventArgs)> _inputSubject = new();
       private readonly List<GridDrawOperation> _drawOps = new();
       private GridViewModel? _parent;
       private bool _busy, _mouseEnabled = true;
       private MouseButton _mousePressedButton = MouseButton.None;
       private GridViewModel _mousePressedVm;
       private (int row, int col) _mousePos;
       private GridSize _gridSize;
       private double _gridScale = 1.0;
       private GridBufferCell[,] _gridBuffer;
       private bool _gridDirty;
       private double _fontSize;
       private Size _glyphSize = new(10, 10);
       private bool _gridFocused, _gridFocusable = true;
       // float/external/message grid state
       private bool _isHidden, _isExternal, _isFloat, _isMsg;
       private Anchor _anchor;
       private int _anchorRow, _anchorCol, _anchorGrid;
   }
   ```

   **Key methods to translate**:
   - `Redraw(cmd)` — large switch on `RedrawCommand` (handles GridLine, GridScroll, GridClear, GridCursorGoto, WinPos, WinFloatPos, WinHide, WinClose, PopupMenu*, Flush, etc.)
   - `ParseGridLine(line)` — fills grid buffer cells from `GridLine` data
   - `ScrollBuffer(top, bot, left, right, rows, cols)` — efficient array memory copy
   - Input event handlers: `OnKeyDown`, `OnKeyUp`, `OnTextInput`, `OnPointerPressed`, `OnPointerReleased`, `OnPointerMoved`, `OnPointerWheelChanged`
   - `CreateChild(id, rows, cols)` / `AddChild` / `RemoveChild` / `Detach`
   - `GetPhysicalSize()` — computes pixel size from glyph size × grid dimensions

7. **`GridDrawOperation` DU** → enum or sealed hierarchy:
   ```csharp
   public abstract record GridDrawOperation;
   public sealed record ScrollOp(int Top, int Bot, int Left, int Right, int Rows, int Cols) : GridDrawOperation;
   public sealed record PutOp(GridRect Rect) : GridDrawOperation;
   ```

8. **`TrackedGridPosition` record**:
   ```csharp
   public class TrackedGridPosition {
       public int TRow;
       public int TCol;
   }
   ```

### FrameViewModel.fs

9. **Translate `FrameViewModel`**: Implements `IFrame`. Manages window title, background image, composition mode, custom title bar:
   ```csharp
   public class FrameViewModel : ThemableViewModelBase, IFrame {
       private string _title = "FVim";
       private IGridUI _mainGrid;
       private BackgroundComposition _composition;
       // ...
       public string Title { get => _title; set => this.RaiseAndSetIfChanged(ref _title, value); }
       public IGridUI MainGrid => _mainGrid;
       public void Sync(RedrawCommand cmd) => _mainGrid.Redraw(cmd);
   }
   ```

### TitleBarViewModel.fs, CrashReportViewModel.fs

10. **Translate remaining simple ViewModels** — these are small and straightforward.

## References

- Source: `ViewModels/*.fs`
- ReactiveUI C# docs: https://www.reactiveui.net/docs/handbook/
- `ObservableCollection<T>`: use for popup menu items (supports data binding change notifications)
- Avalonia data binding: https://docs.avaloniaui.net/docs/data-binding
