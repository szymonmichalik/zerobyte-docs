# SDKs

Four SDKs implement the client side of Relay Protocol v1. They share one design, so the concepts on
this page apply to all of them.

| SDK | Platforms | Crypto | Dependencies | Page |
|---|---|---|---|---|
| Swift | iOS 16, macOS 13, watchOS 9, tvOS 16, visionOS 1 | CryptoKit | None | [swift.md](swift.md) |
| Kotlin | Android API 28, any JVM 11 | `javax.crypto` | Kotlin standard library | [android.md](android.md) |
| JavaScript | Browsers, Node 20, Deno, Bun, Cloudflare Workers | WebCrypto, `@noble/ciphers` | `@noble/ciphers` | [javascript.md](javascript.md) |
| .NET | .NET 10 on Windows 10 20H2+, Linux, macOS | `System.Security.Cryptography` | None | [dotnet.md](dotnet.md) |

## Layers

Each SDK has the same four layers. Most apps use only `Credentials`, `Relay` and `Link`.

| Layer | Does |
|---|---|
| `Credentials` | Derives the mailbox id, bearer token, token digest and the two direction keys from the link key and link id |
| `Envelope` | Seals and opens one envelope; reads the clear header |
| `Wire` | Joins and splits the fetch response body |
| `Relay` | Talks HTTPS to a relay; implements the transport interface; opens the watch WebSocket |
| `Link` | One side of a paired link: sequence numbers, claiming, replay protection, receiving and pruning |

The transport is an interface (`RelayTransport`, `IRelayTransport` in .NET). `Relay` implements it
over HTTPS; you can implement it in memory for tests, or over another channel.

## The pattern in every app

```
on pairing:
    store link key, link id and role in the platform keystore

on launch:
    credentials = Credentials(link key, link id)
    relay       = Relay(app id, credentials, base = your relay's /v1/ URL)
    link        = Link(credentials, sends = your direction, relay, lastReceived = stored value)

to send:
    link.send(bytes)

while in the foreground:
    watch(direction you read):
        on open and on each message:
            for message in link.receive(): handle it
            store link.lastReceived

on unpair:
    link.erase()
    delete link key, link id, role and lastReceived
```

## Behaviour common to every `Link`

- **Claims lazily.** The first `send` or `receive` claims the mailbox. You never call `claim`.
- **Picks sequence numbers.** The first `send` in a process asks the relay for the last sequence
  (`HEAD`), then counts up. On `409` it continues after the relay's last sequence with a new nonce,
  up to three attempts, then throws `sequence_conflict`.
- **Receives everything.** `receive` fetches page after page until the relay has nothing more,
  opens each envelope, skips those that fail to open, prunes everything it saw, and returns the
  opened messages in order with their sequence numbers.
- **Protects against replays.** It refuses any envelope at or below `lastReceived`. Pass the stored
  value back when you construct the `Link`.
- **Serialises its operations.** Calls on one `Link` run one at a time (Swift actor, Kotlin
  `synchronized`, JavaScript promise chain, .NET semaphore), so two sends never race for a
  sequence.
- **Erases for both sides.** `erase` deletes the mailbox; the next operation claims afresh.

## Errors

Two error types, with the same codes in every SDK.

**Envelope errors** (`ZeroByteError` in Swift and JavaScript, `ZeroByteException` in Kotlin and
.NET):

| Code | Meaning |
|---|---|
| `invalid_key` | Link key not 32 bytes, or link id not 16 |
| `malformed` | Envelope shorter than 38 bytes, or direction byte not 1 or 2 |
| `unsupported_version` | Version byte not 1 |
| `wrong_direction` | Envelope written in the other direction |
| `replayed` | Sequence at or below the last accepted |
| `authentication_failed` | Altered, or sealed under another link's keys |
| `too_large` | Plaintext over 1 MiB |

**Relay errors** (`RelayError`, `RelayException`):

| Code | HTTP | Meaning |
|---|---|---|
| `unauthorized` | 401, 403 | Mailbox not claimed, or owned by another token |
| `sequence_conflict` | 409 | Sequence not above the last; carries the relay's last sequence |
| `quota_exceeded` | 413, 507 | Envelope too large or mailbox full |
| `unavailable` | other | Any other status; carries it |
| `malformed_response` | 2xx | The fetch body could not be parsed |

In Swift the codes are enum cases: `invalidKey`, `malformedEnvelope`, `unsupportedVersion`,
`wrongDirection`, `replayed`, `authenticationFailed`, `tooLarge`; and `unauthorized`,
`sequenceConflict(last:)`, `quotaExceeded`, `unavailable(status:)`, `malformedResponse`.

`Link.receive` never throws envelope errors: an envelope that fails to open is skipped and pruned.
It throws relay errors and network errors.

## Installing

None of the SDKs is published to a public package registry yet. Each page shows how to install
from the GitHub repository at tag `v1.0.0`.

## Base URL

Every `Relay` takes the base URL of your relay, ending in `/v1/`. It is required: the SDKs have no
default relay.
