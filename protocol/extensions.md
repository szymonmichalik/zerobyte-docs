# Extensions

Optional features a relay operator turns on per app in [`RELAY_APPS`](../relay/configuration.md).
None of them changes a byte of the keys or the envelope. A client must work without them. A
path the app has not enabled answers the same `404` as any unknown path.

| Extension | Option | Gives |
|---|---|---|
| [Rate limits](#rate-limits) | Always on | `429` with `Retry-After` instead of unbounded load |
| [Heartbeat claim](#heartbeat-claim) | Always on | Keeps a quiet mailbox alive without writing |
| [Push](#push) | `push=<topic>` | Wakes the reader through a push service (APNs module) when a message arrives |
| [Signal](#signal) | `signal` | A rendezvous for a direct device-to-device connection |
| [ICE](#ice) | `ice` | Short-lived STUN/TURN credentials for that connection |

## Rate limits

Each mailbox has token buckets, kept in memory only:

| Bucket | Burst | Refill |
|---|---|---|
| Envelopes stored | 120 | 2 per second |
| Fetches, `HEAD`s, prunes | 240 | 4 per second |
| WebSocket connections (watch and signal) | 20 | 1 per 5 seconds |
| Signal frames, per side | 400 | 50 per second |
| Pushes sent, per direction | 20 | 1 per 2 seconds |
| Push registrations | 10 | 1 per minute |
| ICE credentials | 6 | 1 per 30 seconds |

A request over a limit answers:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
Content-Type: application/json

{"error":"rate_limited","status":429}
```

Nothing was stored. Wait `Retry-After` seconds and retry the same request. A signal socket that
sends too fast is closed with `1008`. Pushes over their limit are skipped silently: the write
still succeeds.

## Heartbeat claim

A `PUT` claim that answers `204` (already yours) restarts the idle timer, like a write. An app
whose link may stay quiet for longer than the idle time can claim once a week to keep the
mailbox.

## Push

The relay can wake the reading device when the writer asks it to. It sends a push with no content:
the device fetches and decrypts the message itself.

The relay core knows no push service. A registration names its `service`; a separate module of
the relay checks that service's fields and delivers. The relay ships two modules, **APNs** for
Apple platforms and **FCM** for Android, and loads each only when the operator configures it. A
dedicated app can add its own module (a UnifiedPush gateway, for example) without changing the
protocol: the generic members `service`, `tag` and `alert` stay the same.

### Register

The reader registers the device on the direction it **reads** (the companion on `p2c`, the
primary on `c2p`), after every launch, because device tokens change:

```http
PUT /v1/<app>/mailbox/<id>/p2c/push
Authorization: Bearer <token>
Content-Type: application/json

{"service":"apns","token":"<device token, hex>","environment":"production",
 "tag":"Zq3k8r2mT1u0wQ","alert":"New message"}
```

On Android:

```json
{"service":"fcm","token":"<FCM registration token>","tag":"Zq3k8r2mT1u0wQ","alert":"New message"}
```

| Field | Rules |
|---|---|
| `service` | `apns` or `fcm`; in general 1 to 16 of `a-z 0-9 -`, starting with a letter |
| `token` | APNs: the device token, lowercase hex, 32 to 100 bytes. FCM: the registration token, 32 to 1024 of `A-Z a-z 0-9 _ : -` |
| `environment` | APNs only: `production`, or `development` for debug builds |
| `tag` | 1 to 64 of `A-Z a-z 0-9 _ -`. Comes back in the push so the app can find the link. Make it random per link: Apple or Google sees it. |
| `alert` | Optional, at most 160 characters: the text shown if the device cannot decrypt in time |

Answers `204`, or `400` for anything else, including fields the service's module does not know.
A relay without a module for the service keeps the registration (other fields: at most 8
strings of at most 256 characters) and sends nothing. `DELETE …/p2c/push` removes the
registration. The relay also removes it when APNs answers `410` or `BadDeviceToken`, or FCM
answers `UNREGISTERED`.

### Ask for a push

The writer adds a header to the `PUT` of an envelope:

| `X-Relay-Notify` | Push |
|---|---|
| (absent) | None |
| `alert` | A visible notification with `mutable-content: 1`, priority 10 |
| `silent` | A background push (`content-available: 1`), priority 5 |

Any other value answers `400`. The push's payload is only:

```json
{"aps":{"alert":{"body":"New message"},"mutable-content":1,"sound":"default"},
 "zerobyte":{"tag":"Zq3k8r2mT1u0wQ"}}
```

No mailbox id, no sequence, no content. On iOS, a Notification Service Extension looks the tag
up, fetches and opens the envelope, and replaces the body with the message. Pushes are
best-effort; the app still fetches when it launches.

On Android every FCM message is data-only, so `FirebaseMessagingService.onMessageReceived` runs
whether the app is in the foreground or not, and the app draws the notification itself:

```json
{"message":{"token":"<registration token>",
  "data":{"zerobyte_tag":"Zq3k8r2mT1u0wQ","zerobyte_notify":"alert","zerobyte_alert":"New message"},
  "android":{"priority":"high","ttl":"604800s"}}}
```

A silent push has `zerobyte_notify: silent`, no `zerobyte_alert`, normal priority and a one-hour
lifetime.

The relay sends no push while the reader's [watch](watch.md) socket has sent `ping` within the
last 45 seconds: the app is in the foreground and already knows.

### What the relay stores

The registration is sealed with AES-256-GCM under

```
HKDF-SHA256(ikm = access token, salt = "zerobyte-push-v1", info = "<mailbox id>/<p2c|c2p>", 32 bytes)
```

The relay keeps only the token's SHA-256, so it can open a registration only while it serves a
request that presents the bearer: a write with `X-Relay-Notify`. A copy of its database holds no
device token and cannot link two mailboxes of one device.

### Operator setup

For the APNs module: an APNs auth key (`.p8`) from the Apple Developer account, with its key id
and the team id. See [configuration](../relay/configuration.md#push-and-ice-settings). Without
those settings the module is not loaded and `X-Relay-Notify` is accepted and ignored.

## Signal

```
GET /v1/<app>/mailbox/<id>/signal/<primary|companion>
```

A WebSocket, upgraded exactly like [watch](watch.md) (subprotocol `zerobyte.v1`, bearer in the
header or as `zerobyte.bearer.<token>`). Each device opens its own side.

| Relay sends | When |
|---|---|
| Text `peer 1` | To both sides, when both are connected |
| Text `peer 0` | To the side left, when the other disconnects |
| A binary frame | Exactly as the other side sent it |
| Text `pong` | In answer to `ping` |

| Client sends | Relay does |
|---|---|
| A binary frame of at most 64 KiB | Forwards it to the other side, if connected; otherwise drops it |
| Text `ping` | Answers `pong` |
| Any other text | Closes with `1003` |
| A larger frame | Closes with `1009` |

A second socket for the same side replaces the first, which is closed with `4000 replaced`.
Erasing the mailbox closes both with `1000`.

The relay stores nothing and reads nothing. Frames must be sealed by the devices; the SDKs seal
them like envelopes, under keys of their own:

| Value | `info` | Length |
|---|---|---|
| Signal key p2c | `zerobyte-signal-p2c-v1` | 32 |
| Signal key c2p | `zerobyte-signal-c2p-v1` | 32 |

HKDF-SHA256 with the link key and salt = link id, as in [Keys](keys.md). The frame layout is the
envelope's, with the sequence replaced by a counter that is milliseconds since 1970, made strictly
increasing. A receiver drops a frame whose counter is at or below the last it accepted, or more
than five minutes from its own clock.

Typical use: both devices open their side; on `peer 1` the primary sends a WebRTC offer, the
companion answers, both send ICE candidates; once the data channel is up, both close the signal
socket and talk directly.

## ICE

```http
GET /v1/<app>/mailbox/<id>/ice
Authorization: Bearer <token>
```

```json
{"iceServers":[
  {"urls":["stun:stun.example.com:3478"]},
  {"urls":["turn:turn.example.com:3478?transport=udp"],
   "username":"1767225600:3f2a…","credential":"base64…"}],
 "ttl":3600}
```

The shape of WebRTC's `RTCIceServer`. Credentials expire after `ttl` seconds; ask again before a
new connection. `502 upstream_unavailable` when the TURN provider could not be reached.

The relay can hand out credentials for coturn (the REST API scheme with a shared secret) and for
Cloudflare Realtime TURN, or both. A TURN server relays the direct connection when no direct path
exists. It sees both devices' addresses and the size and timing of the traffic, never its
content: the connection is encrypted end to end (DTLS for WebRTC, plus whatever the app adds).

## What the extensions add to what the relay sees

- When a signal frame passes and its size, and both devices' addresses while their sockets are
  open.
- Whether the reader is in the foreground (its watch socket pings).
- The APNs device token, only during a request that presents the bearer.

Apple sees the device token, the topic (bundle id), the time of each push and the tag.
