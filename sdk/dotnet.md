# .NET SDK

[zerobyte-dotnet-sdk](https://github.com/szymonmichalik/zerobyte-dotnet-sdk). Apache-2.0. Package
id `ZeroByte`.

| | |
|---|---|
| Target | .NET 10 (`net10.0`): WinUI, WPF, Windows Forms, MAUI, ASP.NET Core, worker services, console |
| Uses | `System.Security.Cryptography` (HKDF, SHA-256, ChaCha20-Poly1305), `HttpClient`, `ClientWebSocket` |
| Dependencies | None |

## Install

The package is not on NuGet yet. Reference the project directly, or build a package and add it
from a local folder:

```sh
git clone --branch v1.0.0 https://github.com/szymonmichalik/zerobyte-dotnet-sdk.git
dotnet pack zerobyte-dotnet-sdk/src/ZeroByte -o ./packages
dotnet add package ZeroByte --source ./packages
```

Or, as a project reference:

```sh
dotnet add reference ../zerobyte-dotnet-sdk/src/ZeroByte/ZeroByte.csproj
```

## Platform support

ChaCha20-Poly1305 comes from the operating system. It is available on Windows 10 20H2 and newer,
on Linux with OpenSSL 1.1 or newer, and on macOS. Check before use:

```csharp
if (!Envelope.IsSupported)
    throw new PlatformNotSupportedException("ChaCha20-Poly1305 is not available on this system.");
```

Every `Envelope` call throws on a system without it.

## Set up a link

```csharp
using ZeroByte;

var relayBase = new Uri("https://relay.example.com/v1/");
const string AppId = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d";

var credentials = new Credentials(linkKey, linkId);              // byte[32], Guid
var relay = new Relay(AppId, credentials, relayBase);
var link = new Link(credentials, Direction.C2P, relay, store.LastReceived);   // ulong?
```

`Credentials` throws `ZeroByteException` with `Code == "invalid_key"` if the key is not 32 bytes or
the id not 16. A `Guid` is read in RFC 4122 byte order (`ToByteArray(bigEndian: true)`), which
matches the UUID's text form and the other SDKs. Passing `guid.ToByteArray()` as bytes instead
would produce a different link id.

## Send

```csharp
ulong sequence = await link.SendAsync("hello"u8.ToArray());
```

Throws `ZeroByteException("too_large")` above 1 MiB and `RelayException` for relay failures.
Network failures surface as `HttpRequestException`.

## Receive

```csharp
async Task PullAsync(CancellationToken ct)
{
    foreach (var message in await link.ReceiveAsync(ct))
        Handle(message.Sequence, message.Plaintext);
    store.LastReceived = link.LastReceived;
}
```

`ReceiveAsync` returns `List<Opened>`; `Opened` is a `record struct` with `Sequence` and
`Plaintext`.

## Watch

```csharp
async Task WatchAsync(CancellationToken ct)
{
    var delay = TimeSpan.FromSeconds(1);
    while (!ct.IsCancellationRequested)
    {
        try
        {
            await relay.WatchAsync(
                Direction.P2C,                                   // the direction you read
                onSequence: _ => _ = PullAsync(ct),
                onOpen: () => { delay = TimeSpan.FromSeconds(1); _ = PullAsync(ct); },
                ct: ct);
        }
        catch (Exception) when (!ct.IsCancellationRequested) { }
        await Task.Delay(delay, ct);
        delay = TimeSpan.FromSeconds(Math.Min(delay.TotalSeconds * 2, 60));
    }
}
```

`WatchAsync` connects, calls `onOpen` once the socket is open, calls `onSequence` for each
announced sequence, and returns when the socket closes or `ct` is cancelled. It authenticates with
the `Authorization` header and offers `zerobyte.v1`.

`Link` runs one operation at a time, so overlapping `PullAsync` calls are safe. In a UI app,
marshal `Handle` back to the UI thread.

## Unpair

```csharp
await link.EraseAsync();
DeleteStoredLink();
store.LastReceived = null;
```

## Store the link key

On Windows, protect the link key with DPAPI for the current user:

```csharp
using System.Security.Cryptography;

byte[] protectedKey = ProtectedData.Protect(linkKey, optionalEntropy: null,
    DataProtectionScope.CurrentUser);
File.WriteAllBytes(path, protectedKey);

byte[] linkKey = ProtectedData.Unprotect(File.ReadAllBytes(path), null,
    DataProtectionScope.CurrentUser);
```

`ProtectedData` is in the `System.Security.Cryptography.ProtectedData` package. On macOS use the
Keychain, on Linux the Secret Service (libsecret). See [Storing the link key](../guides/key-storage.md).

## Sharing an `HttpClient`

`Relay` creates its own `HttpClient` with redirects and cookies disabled. To share one, for example
from `IHttpClientFactory`, pass it in and configure it the same way:

```csharp
services.AddHttpClient("zerobyte")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        AllowAutoRedirect = false,
        UseCookies = false,
    });

var relay = new Relay(AppId, credentials, relayBase, factory.CreateClient("zerobyte"));
```

## API reference

```csharp
public enum Direction : byte { P2C = 1, C2P = 2 }
public static class DirectionExtensions { public static Direction Reversed(this Direction d); }

public sealed class Credentials
{
    public Credentials(ReadOnlySpan<byte> linkKey, ReadOnlySpan<byte> linkID);
    public Credentials(ReadOnlySpan<byte> linkKey, Guid linkID);
    public byte[] MailboxID { get; }          // 16 bytes
    public byte[] AccessToken { get; }        // 32 bytes
    public string MailboxName { get; }        // 32 lowercase hex characters
    public string BearerToken { get; }        // 43 characters, base64url
    public string TokenDigest { get; }        // 64 lowercase hex characters
}

public sealed class Link
{
    public Link(Credentials credentials, Direction sends, IRelayTransport relay,
                ulong? lastReceived = null);
    public Direction Sends { get; }
    public Direction Receives { get; }
    public ulong? LastReceived { get; }
    public Task<ulong> SendAsync(byte[] plaintext, CancellationToken ct = default);
    public Task<List<Opened>> ReceiveAsync(CancellationToken ct = default);
    public Task EraseAsync(CancellationToken ct = default);
}

public readonly record struct Opened(ulong Sequence, byte[] Plaintext);

public sealed class Relay : IRelayTransport
{
    public Relay(string appID, Credentials credentials, Uri @base, HttpClient? http = null);
    public Uri Mailbox { get; }
    public Task WatchAsync(Direction direction, Action<ulong> onSequence, Action? onOpen = null,
                           CancellationToken ct = default);
    // IRelayTransport methods
}

public interface IRelayTransport
{
    Task ClaimAsync(CancellationToken ct = default);
    Task PutAsync(byte[] envelope, Direction direction, ulong sequence, CancellationToken ct = default);
    Task<Page> FetchAsync(Direction direction, ulong? after, CancellationToken ct = default);
    Task<ulong?> LastSequenceAsync(Direction direction, CancellationToken ct = default);
    Task PruneAsync(Direction direction, ulong through, CancellationToken ct = default);
    Task EraseAsync(CancellationToken ct = default);
}

public sealed record Page(IReadOnlyList<byte[]> Envelopes, bool More);

public static class Envelope
{
    public const int Overhead = 38;
    public const int MaxPlaintextBytes = 1_048_576;
    public const int MaxEnvelopeBytes = MaxPlaintextBytes + Overhead;
    public static bool IsSupported { get; }
    public static byte[] Seal(ReadOnlySpan<byte> plaintext, Direction direction, ulong sequence,
                              Credentials credentials, byte[]? nonce = null);
    public static Opened Open(ReadOnlySpan<byte> envelope, Direction direction, ulong? after,
                              Credentials credentials);
    public static (Direction Direction, ulong Sequence) ReadHeader(ReadOnlySpan<byte> envelope);
}

public static class Wire
{
    public const int MaximumCount = 64;
    public static byte[] Join(IReadOnlyList<byte[]> envelopes);
    public static List<byte[]> Split(ReadOnlySpan<byte> body);
}

public sealed class ZeroByteException : Exception { public string Code { get; } }
public sealed class RelayException : Exception
{
    public string Code { get; }
    public int Status { get; }
    public ulong? Last { get; }
}
```

The `nonce` parameter of `Envelope.Seal` is for known-answer tests only. Leave it out.

## Testing

```sh
dotnet test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> dotnet test
```

On macOS with Homebrew's .NET, set `DOTNET_ROOT=/opt/homebrew/opt/dotnet/libexec` if the test host
cannot find the runtime.
