# Swift SDK

[zerobyte-swift-sdk](https://github.com/szymonmichalik/zerobyte-swift-sdk). Apache-2.0.

| | |
|---|---|
| Platforms | iOS 16, iPadOS 16, macOS 13, watchOS 9, tvOS 16, visionOS 1 |
| Swift | 6, strict concurrency |
| Uses | CryptoKit (HKDF, SHA-256, ChaCha20-Poly1305), URLSession (HTTPS, WebSocket) |
| Dependencies | None |

## Install

In Xcode: File > Add Package Dependencies, enter
`https://github.com/szymonmichalik/zerobyte-swift-sdk.git`, and choose version 1.0.0 or later.

In `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/szymonmichalik/zerobyte-swift-sdk.git", from: "1.0.0")
],
targets: [
    .target(name: "MyApp", dependencies: [
        .product(name: "ZeroByte", package: "zerobyte-swift-sdk")
    ])
]
```

## Set up a link

```swift
import ZeroByte

let relayBase = URL(string: "https://relay.example.com/v1/")!
let appID = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d"

let credentials = try Credentials(linkKey: linkKey, linkID: linkID)   // Data (32 bytes), UUID
let relay = Relay(appID: appID, credentials: credentials, base: relayBase)
let link = Link(
    credentials: credentials,
    sends: .primaryToCompanion,          // .companionToPrimary on the other device
    transport: relay,
    lastReceived: store.lastReceived)    // UInt64?, nil on a new link
```

`Credentials(linkKey:linkID:)` throws `ZeroByteError.invalidKey` if the key is not 32 bytes or the
id not 16. The `linkID` may be a `UUID` or 16 bytes of `Data`.

## Send

```swift
let sequence = try await link.send(Data("hello".utf8))
```

`send` is `@discardableResult` and returns the sequence the message was stored under. It throws
`ZeroByteError.tooLarge` above 1 MiB, and `RelayError` for relay failures.

## Receive

```swift
for message in try await link.receive() {
    handle(message.plaintext)          // message.sequence is the UInt64 sequence
}
store.lastReceived = await link.lastReceived
```

`receive()` returns `[(sequence: UInt64, plaintext: Data)]` in order. Save `lastReceived` after you
have handled the messages.

## Watch

```swift
func watch() async {
    var delay: Duration = .seconds(1)
    while !Task.isCancelled {
        let notifications = relay.watch(direction: .companionToPrimary)   // opens the socket
        await pull()                                    // catch up on (re)connect
        for await _ in notifications {
            delay = .seconds(1)
            await pull()
        }
        try? await Task.sleep(for: delay)               // the socket closed: back off
        delay = min(delay * 2, .seconds(60))
    }
}

func pull() async {
    do {
        for message in try await link.receive() { handle(message.plaintext) }
        store.lastReceived = await link.lastReceived
    } catch {
        // network or relay error; the next notification or reconnect retries
    }
}
```
starts the socket and returns an `AsyncStream<UInt64>` of sequence
numbers. The stream ends when the socket closes; cancelling the consuming task closes the socket.
Watch the direction you **read**, the reverse of `sends`.

`URLSessionWebSocketTask` does not report the moment the handshake completes, so the stream has no
"open" event. The `pull()` right after starting the watch covers almost everything; an envelope
stored in the short window before the handshake completes is picked up by the next notification.
If your app may sit idle with the socket open and nothing else being sent, also pull on a timer
while watching, for example every few minutesng the consuming task closes the socket. Watch the direction you
**read**, the reverse of `sends`.

Run the watch while the app is active. On iOS, cancel the task when the scene goes to the
background and start it again on `.active`:

```swift
.onChange(of: scenePhase) { _, phase in
    if phase == .active { watchTask = Task { await watch() } }
    else { watchTask?.cancel() }
}
```

## Unpair

```swift
try await link.erase()
try keychain.deleteLink()
store.lastReceived = nil
```

## Store the link key

Keep the link key in the Keychain as a generic password, available after first unlock, on this
device only, not synchronised:

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: "com.example.app.zerobyte",
    kSecAttrAccount as String: linkID.uuidString,
    kSecValueData as String: linkKey,
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
    kSecAttrSynchronizable as String: false,
]
let status = SecItemAdd(query as CFDictionary, nil)
```

`ThisDeviceOnly` keeps the key out of iCloud Keychain and out of backups restored to another
device. See [Storing the link key](../guides/key-storage.md).

## API reference

### `Credentials`

```swift
public struct Credentials: Sendable, Hashable {
    public init(linkKey: Data, linkID: Data) throws
    public init(linkKey: Data, linkID: UUID) throws
    public let mailboxID: Data            // 16 bytes
    public let accessToken: Data          // 32 bytes
    public var mailboxName: String        // 32 lowercase hex characters
    public var bearerToken: String        // 43 characters, base64url
    public var tokenDigest: String        // 64 lowercase hex characters
}
```

### `Direction`

```swift
public enum Direction: UInt8 {
    case primaryToCompanion = 1
    case companionToPrimary = 2
    public var pathComponent: String      // "p2c" or "c2p"
    public var reversed: Direction
}
```

### `Link`

```swift
public actor Link {
    public init(credentials: Credentials, sends: Direction, transport: any RelayTransport,
                lastReceived: UInt64? = nil)
    public let credentials: Credentials
    public let sends: Direction
    public private(set) var lastReceived: UInt64?
    @discardableResult public func send(_ plaintext: Data) async throws -> UInt64
    public func receive() async throws -> [(sequence: UInt64, plaintext: Data)]
    public func erase() async throws
}
```

### `Relay`

```swift
public struct Relay: RelayTransport {
    public static let fetchLimit = 16
    public init(appID: String, credentials: Credentials, base: URL)
    public init(endpoint: URL, credentials: Credentials)   // full https://host/v1/<app id>/
    public let endpoint: URL
    public func watch(direction: Direction) -> AsyncStream<UInt64>
    // RelayTransport methods below
}
```

`Relay` uses an ephemeral `URLSession` with no cookies, no cache and no credential store, a 30
second request timeout, and `waitsForConnectivity = false`, so a request without a network fails
fast instead of waiting.

### `RelayTransport`

```swift
public protocol RelayTransport: Sendable {
    func claim() async throws
    func put(_ envelope: Data, direction: Direction, sequence: UInt64) async throws
    func fetch(direction: Direction, after sequence: UInt64?) async throws
        -> (envelopes: [Data], more: Bool)
    func lastSequence(direction: Direction) async throws -> UInt64?
    func prune(direction: Direction, through sequence: UInt64) async throws
    func erase() async throws
}
```

### `Envelope` and `Wire`

```swift
public enum Envelope {
    public static let version: UInt8                  // 1
    public static let overhead: Int                   // 38
    public static let maximumPlaintextBytes: Int      // 1_048_576
    public static let maximumEnvelopeBytes: Int
    public static func seal(_ plaintext: Data, direction: Direction, sequence: UInt64,
                            credentials: Credentials) throws -> Data
    public static func open(_ envelope: Data, direction: Direction, after lastSequence: UInt64?,
                            credentials: Credentials) throws -> (sequence: UInt64, plaintext: Data)
    public static func header(of envelope: Data) throws -> (direction: Direction, sequence: UInt64)
}

public enum Wire {
    public static let maximumCount = 64
    public static func join(_ envelopes: [Data]) -> Data
    public static func split(_ data: Data) throws -> [Data]
}
```

### Errors

```swift
public enum ZeroByteError: Error {
    case invalidKey, malformedEnvelope, unsupportedVersion(UInt8), wrongDirection,
         replayed(sequence: UInt64), authenticationFailed, tooLarge
}

public enum RelayError: Error {
    case unauthorized, sequenceConflict(last: UInt64?), quotaExceeded,
         unavailable(status: Int), malformedResponse
}
```

## Testing

```sh
swift test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> swift test   # also against a live relay
```

In your own tests, implement `RelayTransport` in memory and pass it to two `Link`s. See
[Testing your integration](../guides/testing.md).
