# 05 — Migrate Command-Line Parsing (`getopt.fs`)

**Status:** pending

## Overview

Translate `getopt.fs` to `GetOpt.cs`. This file parses FVim's custom command-line options and separates them from the Neovim arguments that are passed through. It defines several DUs for connection types and execution intent.

## Motivation

The file uses F# discriminated unions for option types and helper functions (`eat1`, `eat2`, `eat2d`) that consume args list head. The `parseOptions` function uses recursive tail-call pattern matching on a `string list`. C# has no tail-call pattern matching on lists; replace with a sequential `while`/`for` loop over `string[]` with an index pointer.

## Steps

1. **Translate DUs to C# sealed hierarchies** (see also task 03 for conventions):

   ```csharp
   // NeovimRemoteEndpoint
   public abstract record NeovimRemoteEndpoint;
   public sealed record TcpEndpoint(System.Net.IPEndPoint Address) : NeovimRemoteEndpoint;
   public sealed record NamedPipeEndpoint(string Address) : NeovimRemoteEndpoint;

   // FVimRemoteVerb
   public abstract record FVimRemoteVerb;
   public sealed record AttachToVerb(int Pid) : FVimRemoteVerb;
   public sealed record NewSessionVerb(List<string> Args) : FVimRemoteVerb;
   public sealed record AttachFirstVerb : FVimRemoteVerb;

   // FVimRemoteTransport
   public abstract record FVimRemoteTransport;
   public sealed record LocalTransport : FVimRemoteTransport;
   public sealed record RemoteTransport(string Program, List<string> Args) : FVimRemoteTransport;

   // ServerOptions
   public abstract record ServerOptions;
   public sealed record EmbeddedServer(string Program, List<string> Args, System.Text.Encoding StderrEncoding) : ServerOptions;
   public sealed record NeovimRemoteServer(NeovimRemoteEndpoint Endpoint, List<string> Args) : ServerOptions;
   public sealed record FVimRemoteServer(string? PipeName, FVimRemoteTransport Transport, FVimRemoteVerb Verb, List<string> Args) : ServerOptions;

   // Intent
   public enum Intent { Start, Setup, Uninstall, Daemon }

   // Options
   public record Options(
       Intent Intent,
       bool LogToStdout,
       string LogToFile,
       List<string> LogPatterns,
       ServerOptions Server);
   ```

2. **Translate `parseOptions()`**: The F# version is a recursive list-consuming function. In C#, use a `List<string>` queue with a `Dequeue`-style helper, or an index over `args`:
   ```csharp
   public static Options ParseOptions(string[] rawArgs) {
       var args = new Queue<string>(rawArgs);
       var intent = Intent.Start;
       bool logToStdout = false;
       string logToFile = "";
       var logPatterns = new List<string>();
       // FVim-specific flags
       // nvim passthrough args collected separately
       var nvimArgs = new List<string>();
       // ... parse loop consuming from args queue
       // When unrecognized arg: add to nvimArgs
       return new Options(...);
   }
   ```

3. **Translate `eat1` / `eat2` / `eat2d` helpers**: These consume one or two args from the list head. In C#:
   ```csharp
   private static bool Eat1(Queue<string> args, string flag) {
       if (args.TryPeek(out var s) && s == flag) { args.Dequeue(); return true; }
       return false;
   }
   private static bool Eat2(Queue<string> args, string flag, out string value) {
       if (args.TryPeek(out var s) && s == flag) {
           args.Dequeue();
           return args.TryDequeue(out value!);
       }
       value = ""; return false;
   }
   ```

4. **Port the FVim-specific flag list** from `getopt.fs` — search for all `| "--fvim-..."` match arms and translate each to an `else if` branch in the parse loop. Key flags include:
   - `--fvim-no-ext-tabline`, `--fvim-no-ext-popupmenu`, etc. → set states fields
   - `--fvim-log-to-stdout`, `--fvim-log-to-file`, `--fvim-log-pattern`
   - `--fvim-daemon`, `--fvim-setup`, `--fvim-uninstall`
   - `--server`, `--remote`, `--remote-silent`, `--remote-tab`, `--remote-send`
   - `--fvim-ssh`, `--fvim-wsl`

5. **Restore `--` passthrough behavior**: Any arg after `--` or not starting with `--fvim-` should be forwarded to Neovim unchanged.

6. **Add tests**: The option-parsing logic is pure and easy to unit test. Write at least one test per flag group.

## References

- Source: `getopt.fs`
- For IP/DNS parsing: use `System.Net.IPAddress.TryParse` + `Dns.GetHostEntry` (same as F# version)
