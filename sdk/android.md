# Android and JVM SDK

[zerobyte-android-sdk](https://github.com/szymonmichalik/zerobyte-android-sdk). Apache-2.0.

| | |
|---|---|
| Platforms | Android API 28 and newer; any JVM 11 and newer |
| Language | Kotlin, usable from Java |
| Uses | `javax.crypto` (ChaCha20-Poly1305, HmacSHA256), `java.security.MessageDigest`, `HttpURLConnection` |
| Dependencies | Kotlin standard library only |
| Coordinates | `cloud.zerobyte:zerobyte:1.0.0` |

## Install

The library is not on Maven Central yet. Publish it to your local Maven repository:

```sh
git clone --branch v1.0.0 https://github.com/szymonmichalik/zerobyte-android-sdk.git
cd zerobyte-android-sdk
./gradlew :zerobyte:publishToMavenLocal
```

Then in your app's Gradle files:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        mavenLocal()
    }
}

// app/build.gradle.kts
dependencies {
    implementation("cloud.zerobyte:zerobyte:1.0.0")
}
```

For a team or CI, publish the artefact to your own Maven repository instead of `mavenLocal()`, or
include the `zerobyte` module as a Gradle included build.

The app needs the `INTERNET` permission:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

## Set up a link

```kotlin
import cloud.zerobyte.*

const val RELAY_BASE = "https://relay.example.com/v1/"
const val APP_ID = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d"

val credentials = Credentials(linkKey, linkID)          // ByteArray(32), java.util.UUID
val relay = Relay(APP_ID, credentials, RELAY_BASE)
val link = Link(credentials, Direction.C2P, relay, store.lastReceived)   // ULong?
```

`Credentials` throws `ZeroByteException("invalid_key")` if the key is not 32 bytes or the id not
16. The link id may be a `UUID` or a 16-byte `ByteArray`.

## Blocking calls

Every network call is **blocking**. On Android, call them off the main thread, for example on
`Dispatchers.IO`:

```kotlin
suspend fun send(text: String): ULong = withContext(Dispatchers.IO) {
    link.send(text.toByteArray())
}
```

`Link` is thread-safe: concurrent calls run one at a time.

## Send

```kotlin
val sequence: ULong = link.send(bytes)
```

Throws `ZeroByteException("too_large")` above 1 MiB and `RelayException` for relay failures.
Network failures surface as `IOException`; `RelayException` is a subclass of `IOException`.

## Receive

```kotlin
suspend fun pull() = withContext(Dispatchers.IO) {
    for (message in link.receive()) handle(message.sequence, message.plaintext)
    store.lastReceived = link.lastReceived
}
```

`receive()` returns `List<Opened>` in order; `Opened` has `sequence: ULong` and
`plaintext: ByteArray`.

## Watch

Android has no platform WebSocket client, so the SDK gives you the URL and leaves the socket to the
client your app already uses. With OkHttp:

```kotlin
val request = Request.Builder()
    .url(relay.watchURL(Direction.P2C))               // the direction you read
    .header("Authorization", "Bearer ${credentials.bearerToken}")
    .header("Sec-WebSocket-Protocol", "zerobyte.v1")
    .build()

val socket = okHttp.newWebSocket(request, object : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: Response) {
        scope.launch { pull() }
    }
    override fun onMessage(webSocket: WebSocket, text: String) {
        scope.launch { pull() }
    }
    override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
        if (reason == "erased") onUnpairedRemotely() else scheduleReconnect()
    }
    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        scheduleReconnect()
    }
})
```

`watchURL` turns `https://` into `wss://` (and `http://` into `ws://`). OkHttp lets you set the
`Authorization` header on the upgrade. Clients that cannot may offer the subprotocol
`zerobyte.bearer.<bearer token>` instead.

Close the socket in `onStop` and open it again in `onStart`: the `onOpen` pull catches up. Use
exponential backoff in `scheduleReconnect`. For background delivery, schedule a periodic
`WorkManager` job that calls `pull()`.

## Unpair

```kotlin
withContext(Dispatchers.IO) { link.erase() }
keyStore.deleteLink()
store.lastReceived = null
```

## Store the link key

Android Keystore cannot hold an arbitrary 32-byte secret for export. Wrap the link key with a
Keystore AES key and store the wrapped bytes:

```kotlin
val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
val alias = "zerobyte-link-wrap"
if (!keyStore.containsAlias(alias)) {
    KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore").apply {
        init(
            KeyGenParameterSpec.Builder(alias,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .build())
    }.generateKey()
}
val wrapKey = keyStore.getKey(alias, null) as SecretKey

fun wrap(linkKey: ByteArray): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, wrapKey)
    return cipher.iv + cipher.doFinal(linkKey)                // 12-byte IV, then ciphertext
}

fun unwrap(stored: ByteArray): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, wrapKey, GCMParameterSpec(128, stored, 0, 12))
    return cipher.doFinal(stored, 12, stored.size - 12)
}
```

Store the wrapped bytes in app-private storage and exclude them from cloud backup
(`android:dataExtractionRules`). See [Storing the link key](../guides/key-storage.md).

## API reference

### `Credentials`

```kotlin
class Credentials(linkKey: ByteArray, linkID: ByteArray) {
    constructor(linkKey: ByteArray, linkID: UUID)
    val mailboxID: ByteArray          // 16 bytes
    val accessToken: ByteArray        // 32 bytes
    val mailboxName: String           // 32 lowercase hex characters
    val bearerToken: String           // 43 characters, base64url
    val tokenDigest: String           // 64 lowercase hex characters
    companion object { fun uuidBytes(uuid: UUID): ByteArray }
}
```

### `Direction`

```kotlin
enum class Direction(val wire: Byte, val path: String) {
    P2C(1, "p2c"), C2P(2, "c2p");
    val reversed: Direction
    companion object { fun of(wire: Byte): Direction? }
}
```

### `Link`

```kotlin
class Link @JvmOverloads constructor(
    credentials: Credentials, val sends: Direction, relay: RelayTransport,
    lastReceived: ULong? = null,
) {
    val receives: Direction
    var lastReceived: ULong?              // read-only from outside
    fun send(plaintext: ByteArray): ULong
    fun receive(): List<Opened>
    fun erase()
}

class Opened(val sequence: ULong, val plaintext: ByteArray)
```

### `Relay`

```kotlin
class Relay @JvmOverloads constructor(
    appID: String, credentials: Credentials,
    base: String, timeoutMillis: Int = 30_000,
) : RelayTransport {
    val mailbox: String
    fun watchURL(direction: Direction): String
}
```

### `RelayTransport`

```kotlin
interface RelayTransport {
    fun claim()
    fun put(envelope: ByteArray, direction: Direction, sequence: ULong)
    fun fetch(direction: Direction, after: ULong?): Page
    fun lastSequence(direction: Direction): ULong?
    fun prune(direction: Direction, through: ULong)
    fun erase()
}

class Page(val envelopes: List<ByteArray>, val more: Boolean)
```

### `Envelope` and `Wire`

```kotlin
object Envelope {
    const val OVERHEAD = 38
    const val MAX_PLAINTEXT_BYTES = 1_048_576
    const val MAX_ENVELOPE_BYTES = MAX_PLAINTEXT_BYTES + OVERHEAD
    fun seal(plaintext: ByteArray, direction: Direction, sequence: ULong,
             credentials: Credentials): ByteArray
    fun open(envelope: ByteArray, direction: Direction, after: ULong?,
             credentials: Credentials): Opened
    fun readHeader(envelope: ByteArray): Pair<Direction, ULong>
}

object Wire {
    const val MAXIMUM_COUNT = 64
    fun join(envelopes: List<ByteArray>): ByteArray
    fun split(body: ByteArray): List<ByteArray>
}
```

`seal` has an optional `nonce` parameter for known-answer tests only. Leave it out.

### Errors

```kotlin
class ZeroByteException(val code: String) : Exception
class RelayException(val code: String, val status: Int, val last: ULong? = null) : IOException
```

## From Java

The API uses Kotlin's unsigned `ULong` for sequence numbers. Kotlin hides or renames functions and
constructors that take unsigned types as parameters when seen from Java, so parts of `Link` and
`Envelope` are awkward or unavailable there. Call the SDK from Kotlin code in your project, and
expose to Java whatever small surface your app needs.

## Testing

```sh
./gradlew test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> ./gradlew test
```

The tests run on the JVM without an emulator.
