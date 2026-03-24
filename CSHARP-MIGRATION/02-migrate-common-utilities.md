# 02 — Migrate Common Utilities (`common.fs`)

**Status:** pending

## Overview

Translate `common.fs` to `Common.cs`. This file defines type aliases, custom operators, active patterns, async stream helpers, and process-building utilities used throughout the entire codebase.

## Motivation

F# active patterns and custom operators do not exist in C#. Each idiom needs a careful C# equivalent that maintains the same call sites as much as possible:
- `hashmap<'a,'b>` type alias → `using` directive + extension helpers
- `(>>=)` option bind, `(>?=)` result bind → LINQ-style extension methods
- `(?->)` null-safe member access → null-conditional + `ValueTuple` return
- Active patterns like `(|Integer32|_|)`, `(|ObjArray|_|)` → `TryParse`-style static methods
- `read`/`write` async helpers → direct `async` methods
- `ForceBool` active pattern → static method

## Steps

1. **Create `Common.cs`** with namespace `FVim` and `static class Common`:

2. **`hashmap<'a,'b>` type alias**: Replace with a `using` alias at the top of each file that uses it:
   ```csharp
   using HashMap<TKey, TValue> = System.Collections.Generic.Dictionary<TKey, TValue>;
   ```
   Or define a global using in `GlobalUsings.cs`:
   ```csharp
   global using HashMap = System.Collections.Generic.Dictionary<object, object>;
   ```
   Since the F# type is generic `hashmap<'a,'b>`, use concrete `Dictionary<K,V>` at each usage site.

3. **`mkparams1`–`mkparams5`**: Translate to static methods:
   ```csharp
   public static object[] MkParams(object t1) => new[] { t1 };
   public static object[] MkParams(object t1, object t2) => new[] { t1, t2 };
   // etc.
   ```

4. **Parsing active patterns** — translate to `TryParse`-style methods used with `is` patterns or ternary expressions:
   ```csharp
   public static bool TryParseUInt16(string x, out ushort result) => ushort.TryParse(x, out result);
   public static bool TryParseInt32(string x, out int result) => int.TryParse(x, out result);
   public static bool TryParseIp(string x, out System.Net.IPAddress? result) { ... }
   ```
   At usage sites, replace `match s with | ParseInt32 n -> ...` with:
   ```csharp
   if (Common.TryParseInt32(s, out var n)) { ... }
   ```

5. **Type-test active patterns** — translate to `TryMatch` methods:
   ```csharp
   public static bool TryGetObjArray(object x, out object[]? result) {
       if (x is object[] arr) { result = arr; return true; }
       if (x is List<object> list) { result = list.ToArray(); return true; }
       if (x is IEnumerable<object> seq) { result = seq.ToArray(); return true; }
       result = null; return false;
   }
   public static bool TryGetBool(object x, out bool result) { ... }
   public static bool TryGetString(object x, out string? result) { ... }
   public static bool TryGetInteger32(object x, out int result) {
       // handles int32, int16, sbyte, uint16, uint32, byte
   }
   public static bool TryGetFloat(object x, out double result) { ... }
   ```

6. **`ForceBool` active pattern**:
   ```csharp
   public static bool TryForceBool(object x, out bool result) {
       if (x is bool b) { result = b; return true; }
       if (x is string s) {
           if (s is "v:true" or "true") { result = true; return true; }
           if (s is "v:false" or "false") { result = false; return true; }
           if (int.TryParse(s, out var n)) { result = n != 0; return true; }
           result = s.Length > 0; return true;
       }
       result = default; return false;
   }
   ```

7. **Custom operators**:
   - `(>>=)` option bind: Inline at usage sites as `?.` null-conditional or helper extension:
     ```csharp
     public static TResult? Bind<T, TResult>(this T? opt, Func<T, TResult?> f) where T : class => opt is null ? null : f(opt);
     ```
   - `(>?=)` result bind: Inline as `result.IsOk ? f(result.Value) : result`
   - `(?->)` null-safe: Replace `x ?-> f` with `x is null ? ValueTuple? : (ValueTuple?)f(x)` — often just `x?.SomeProp`

8. **`read` / `write` stream helpers**:
   ```csharp
   public static async Task ReadExact(Stream stream, Memory<byte> buf) {
       int total = 0;
       while (total < buf.Length) {
           int n = await stream.ReadAsync(buf.Slice(total));
           if (n == 0) throw new IOException("read");
           total += n;
       }
   }
   public static ValueTask WriteAsync(Stream stream, ReadOnlyMemory<byte> buf) => stream.WriteAsync(buf);
   ```

9. **`toInt32LE` / `fromInt32LE`**:
   ```csharp
   public static int ToInt32LE(byte[] x) => x[0] | (x[1] << 8) | (x[2] << 16) | (x[3] << 24);
   public static byte[] FromInt32LE(int x) => new[] { (byte)x, (byte)(x >> 8), (byte)(x >> 16), (byte)(x >> 24) };
   ```
   Or use `BinaryPrimitives.ReadInt32LittleEndian` / `WriteInt32LittleEndian` from `System.Buffers.Binary`.

10. **`newProcess`**: Translate directly:
    ```csharp
    public static Process NewProcess(string prog, IEnumerable<string> args, Encoding stderrEnc) {
        var escapedArgs = string.Join(" ", EscapeArgs(args));
        var psi = new ProcessStartInfo(prog, escapedArgs) { ... };
        var p = new Process { StartInfo = psi, EnableRaisingEvents = true };
        AppDomain.CurrentDomain.ProcessExit += (_, _) => { try { p.Kill(true); } catch { } };
        return p;
    }
    ```

11. **`escapeArgs` / `join`**: Translate to static helper methods.

12. **`removeAlpha`**: `Color RemoveAlpha(Color x) => new Color(255, x.R, x.G, x.B);`

13. **`notNull`**: Replace inline with `x is not null`.

14. **`_d`** (defaultValue): Replace `_d x opt` with `opt ?? x` or `opt.GetValueOrDefault(x)`.

15. **`run` / `runSync`**:
    ```csharp
    public static void Run(Task t) => Task.Run(() => t);
    public static void RunSync(Task t) => t.ConfigureAwait(false).GetAwaiter().GetResult();
    ```

## References

- Source: `common.fs`
- `BinaryPrimitives` docs: https://learn.microsoft.com/en-us/dotnet/api/system.buffers.binary.binaryprimitives
