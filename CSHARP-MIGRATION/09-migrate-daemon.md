# 09 — Migrate FVR Daemon (`daemon.fs`)

**Status:** pending

## Overview

Translate `daemon.fs` to `Daemon.cs`. This file implements the FVR (FVim Remote) multiplexer: a named pipe server that allows multiple FVim windows to share a single Neovim session. It uses `task { }` computation expressions, `backgroundTask { }`, named pipes, and process management.

## Motivation

F# `task { }` and `backgroundTask { }` computation expressions map directly to C# `async Task` methods. The recursive `daemon()` loop and `serveSession()` pipe relay become `while (true)` async loops. The `Session` record becomes a C# `record class`.

## Steps

1. **Translate the `Session` record**:
   ```csharp
   public record Session(
       int Id,
       NamedPipeServerStream? Server,
       Process Proc,
       Task ExitHandle);
   ```
   Note: The F# `exitHandle: unit Task` is a background task that completes when the session exits.

2. **Translate `newSession()`**:
   ```csharp
   public static async Task<Session> NewSession(string prog, List<string> args) {
       var proc = Common.NewProcess(prog, args, Encoding.UTF8);
       proc.Start();
       var id = proc.Id;
       var exitHandle = proc.WaitForExitAsync();
       return new Session(id, null, proc, exitHandle);
   }
   ```

3. **Translate `attachSession()` / `attachFirstSession()`**:
   - These functions look up sessions by ID or return the first available one.
   - Translate the hashmap lookup and the protocol messages.

4. **Translate `serveSession()`** — the bidirectional pipe relay:
   ```csharp
   public static async Task ServeSession(Stream client, Stream server) {
       var t1 = RelayAsync(client, server);
       var t2 = RelayAsync(server, client);
       await Task.WhenAny(t1, t2);
   }

   private static async Task RelayAsync(Stream from, Stream to) {
       var buf = new byte[4096];
       while (true) {
           int n = await from.ReadAsync(buf);
           if (n == 0) break;
           await to.WriteAsync(buf.AsMemory(0, n));
       }
   }
   ```

5. **Translate `daemon()`** — the infinite accept loop:
   ```csharp
   public static async Task RunDaemon(string pipeName) {
       while (true) {
           var server = new NamedPipeServerStream(pipeName,
               PipeDirection.InOut, NamedPipeServerStream.MaxAllowedServerInstances,
               PipeTransmissionMode.Byte, PipeOptions.Asynchronous);
           await server.WaitForConnectionAsync();
           _ = HandleClientAsync(server); // fire and forget
       }
   }
   ```

6. **Translate `fvrConnect()`** — the client-side session request protocol:
   - Read the F# version carefully. It writes a verb (AttachTo/NewSession/AttachFirst) to the pipe as a simple byte/int protocol.
   - Translate each message type to equivalent C# byte writing.

7. **Translate `getErrorMsg(id)`** — maps negative return codes to error strings.

8. **Translate constants**:
   ```csharp
   public const string DefaultDaemonName = "fvr";
   public static string PipeAddrUnix(string name) => $"/tmp/{name}";
   ```

9. **`backgroundTask { }` → `Task.Run(async () => ...)`**: Any `backgroundTask` blocks become `Task.Run(async () => { ... })` calls to ensure they run on the thread pool.

10. **Handle `CancellationToken`**: The F# daemon loops don't have explicit cancellation. Add `CancellationToken` parameters to support clean shutdown.

## References

- Source: `daemon.fs`
- `NamedPipeServerStream` docs: https://learn.microsoft.com/en-us/dotnet/api/system.io.pipes.namedpipeserverstream
- F# `task {}` = C# `async Task`: identical semantics, direct translation
