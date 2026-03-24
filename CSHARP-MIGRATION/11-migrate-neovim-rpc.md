# 11 — Migrate Neovim RPC (`neovim.fs`)

**Status:** pending

## Overview

Translate `neovim.fs` to `Neovim.cs`. This is the Neovim RPC protocol layer — the most complex I/O-heavy file. It contains the `Nvim` class (389 lines) which manages the subprocess/stream, MessagePack reading loop, request/response correlation, event routing, and RPC method call helpers.

## Motivation

The `Nvim` class is already an F# class (not a module), so the overall structure maps directly to C#. The main challenges are:
- Mutable fields (`let mutable m_notify = ...`) → private fields
- The `NvimIO` DU for connection state → sealed hierarchy (from task 03)
- The MessagePack reader loop uses `task { }` with recursive dispatch
- `FSharp.Control.Reactive` observables → `System.Reactive`
- The `events` property (an observable) is set up during `start()` via `Observable.Create`

## Steps

1. **Create `Neovim.cs`** as `public class Nvim`.

2. **Translate private mutable fields**:
   ```csharp
   private Func<Request, Task> _notify = DefaultNotify;
   private Func<Request, Task<Result<object,object>>> _call = DefaultCall;
   private IObservable<Event>? _events;
   private NvimIO _io = new DisconnectedIO();
   private List<IDisposable> _disposables = new();
   private CancellationTokenSource _cancelSrc = new();
   ```

3. **Translate `createIO(serveropts)`** — the DU match that creates the connection:
   ```csharp
   private NvimIO CreateIO(ServerOptions opts) => opts switch {
       EmbeddedServer(var prog, var args, var enc) => {
           var proc = Common.NewProcess(prog, args, enc);
           proc.Start();
           return new StandaloneIO(proc);
       },
       NeovimRemoteServer(TcpEndpoint(var ipe), _) => {
           var sock = new Socket(ipe.AddressFamily, SocketType.Stream, ProtocolType.Tcp);
           sock.Connect(ipe);
           return new RemoteSessionIO(new NetworkStream(sock, ownsSocket: true));
       },
       NeovimRemoteServer(NamedPipeEndpoint(var addr), _) => {
           var pipe = new NamedPipeClientStream(".", addr, PipeDirection.InOut,
               PipeOptions.Asynchronous, TokenImpersonationLevel.Impersonation);
           pipe.Connect();
           return new RemoteSessionIO(pipe);
       },
       // FVimRemote cases...
       _ => throw new ArgumentException($"Unknown server options: {opts}")
   };
   ```

4. **Translate `start(opts)`**:
   ```csharp
   public void Start(Options opts) {
       if (_io is not DisconnectedIO || _events is not null)
           throw new InvalidOperationException("neovim: already started");
       _io = CreateIO(opts.Server);
       // set up the event observable
       _events = Observable.Create<Event>(observer => {
           var task = ReadLoopAsync(GetStream(_io), observer, _cancelSrc.Token);
           return Disposable.Create(() => _cancelSrc.Cancel());
       }).Publish().RefCount();
   }
   ```

5. **Translate the MessagePack read loop** — reads msgpack arrays from the stream:
   ```csharp
   private async Task ReadLoopAsync(Stream stream, IObserver<Event> observer, CancellationToken ct) {
       var buf = new byte[1024 * 1024]; // 1MB buffer
       while (!ct.IsCancellationRequested) {
           // Read msgpack message from stream
           // Parse [type, id/method, params/result/error]
           // type=0: RPC request → observer.OnNext(new RpcRequestEvent(...))
           // type=1: RPC response → observer.OnNext(new RpcResponseEvent(...))
           // type=2: notification → observer.OnNext(new NotificationEvent(...))
       }
   }
   ```
   The exact framing: Neovim uses a 4-byte big-endian length prefix before each msgpack message (check the F# source to confirm — or it may be raw msgpack array stream). Read the original carefully.

6. **Translate `call()` / `notify()`**:
   ```csharp
   public async Task<Result<object,object>> Call(string method, object[] parameters) {
       var tcs = new TaskCompletionSource<Result<object,object>>();
       int id = Interlocked.Increment(ref _nextId);
       _pending[id] = tcs;
       await SendRpcRequestAsync(id, method, parameters);
       return await tcs.Task;
   }

   public Task Notify(string method, object[] parameters) =>
       SendRpcNotifyAsync(method, parameters);
   ```

7. **Translate grid/UI notification helpers** — functions like `grid_resize`, `ui_attach`, `nvim_input`, etc.:
   ```csharp
   public Task GridResize(int grid, int width, int height) =>
       Notify("nvim_ui_try_resize_grid", Common.MkParams(grid, width, height));

   public Task UiAttach(int width, int height, Dictionary<string, object> options) =>
       Call("nvim_ui_attach", Common.MkParams(width, height, options));
   ```
   List all methods from the F# version and translate each.

8. **Translate `m_disposables` list** — used to track subscriptions:
   ```csharp
   private readonly CompositeDisposable _disposables = new();
   ```

9. **Handle `Disconnect()` / `Dispose()`**:
   ```csharp
   public void Disconnect() {
       _cancelSrc.Cancel();
       _disposables.Dispose();
       switch (_io) {
           case StandaloneIO { Proc: var proc }: proc.Kill(true); proc.Dispose(); break;
           case RemoteSessionIO { Stream: var s }: s.Dispose(); break;
           case TunneledSessionIO { Proc: var proc }: proc.Kill(true); proc.Dispose(); break;
       }
       _io = new DisconnectedIO();
   }
   ```

10. **Translate `default_notify` / `default_call`** — these are the initial no-op delegates:
    ```csharp
    private static Task DefaultNotify(Request r) => Task.CompletedTask;
    private static Task<Result<object,object>> DefaultCall(Request r) =>
        Task.FromResult(Result<object,object>.Err(box("not connected")));
    ```

## References

- Source: `neovim.fs`
- Neovim RPC protocol: https://neovim.io/doc/user/api.html#rpc-api
- MessagePack stream format: https://github.com/msgpack/msgpack/blob/master/spec.md
