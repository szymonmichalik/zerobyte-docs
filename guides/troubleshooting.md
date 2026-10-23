# Troubleshooting

Start by checking the relay from the command line, with the base URL and app id your app uses:

```sh
BASE=https://relay.example.com/v1/
APP=019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d
curl -si "${BASE}${APP}/mailbox/00000000000000000000000000000000/p2c" -X HEAD \
  -H "Authorization: Bearer AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
```

| Answer | Meaning |
|---|---|
| `401` | The relay is up, the app id is registered, the path is right. The mailbox is not claimed, as expected. |
| `404` | Wrong base URL (missing `/v1/`, extra path), or the app id is not in `RELAY_APPS`. |
| Connection refused, TLS error | The relay or its proxy is down, or the certificate is wrong. |

## Errors by status

### `401 unauthorized`

- **The mailbox is not claimed.** The SDKs claim before the first request in each process. If you
  call the HTTP API yourself, claim first.
- **The other device erased the mailbox** after this process claimed it. Usually an unpair. See
  [Link lifecycle](link-lifecycle.md#unpair).
- **Different credentials on the two devices.** Each derives a different mailbox, so each works
  alone and nothing arrives. Compare `credentials.mailboxName` on both devices (it is not secret).
  Usual causes: the link key or link id was encoded differently on each side (hex on one,
  base64 on the other), or a .NET client passed `Guid.ToByteArray()` bytes; see below.
- **A missing or malformed `Authorization` header**, for example a proxy that strips it.

### `403 forbidden` on claim

Another token owns the mailbox. With correctly derived credentials that needs the same mailbox id
under a different token, which cannot happen by chance. Check that both devices derive the token
from the same link key, and that your own HTTP code sends the bearer of this link.

### `404 not_found`

- The base URL does not end in `/v1/`. `https://relay.example.com/` and
  `https://relay.example.com/v1` (no slash, in SDKs that resolve relative URLs) both miss.
- The app id is not registered on this relay, or has a typo, or is uppercase. App ids are lowercase
  UUIDv7.
- A sequence of 0, or with leading zeros, in a hand-built URL.

The relay deliberately answers all of these the same way, so a `404` does not say which.

### `409 sequence_conflict`

Another writer on the same side stored that sequence first: a second install of the same role, or
an earlier process. The SDKs retry from `X-Relay-Last` + 1 up to three times. If it keeps failing,
two devices are probably both acting as primary (or both as companion). Check the stored roles.

### `413` or `507`: `quota_exceeded`

- `413`: one message is above 1 MiB. Split it. If messages well below 1 MiB get `413`, a proxy in
  front of the relay has a smaller body limit (nginx defaults to 1 MiB; set `client_max_body_size 2m`).
- `507`: the direction holds 64 envelopes, or the mailbox 32 MiB. The receiver has not collected.
  Do not retry in a loop: wait until it does, or coalesce. See [Designing messages](message-design.md).

### `5xx` other than `507`

The relay or its proxy failed. Retry with exponential backoff. For a relay you run, see its logs
and [Operating a relay](../relay/operations.md).

## Nothing arrives, no error

1. **Roles.** One device must send `c2p` and the other `p2c`. If both send the same direction, each
   reads a direction nobody writes.
2. **Same relay and app id** on both devices.
3. **`lastReceived` ahead of the sender.** After a mailbox reset, the receiver refuses everything
   up to its old `lastReceived`. See [The mailbox was reset](link-lifecycle.md#the-mailbox-was-reset).
4. **Watching the wrong direction.** Watch the direction you **read**, not the one you send.
5. **No pull after the watch opens.** Envelopes stored while the socket was closed are not
   announced. Pull when it opens.
6. **Forged or damaged envelopes** are skipped silently by design. If every envelope is skipped,
   the two devices have different keys; compare `mailboxName` first, then re-pair.

## Messages arrive twice

The app handled messages and was killed before saving `lastReceived`; the next run fetched them
again. Save `lastReceived` together with the data, or make handling idempotent. The protocol
never delivers the same envelope twice to a receiver that kept its `lastReceived`.

## Watch problems

| Symptom | Cause |
|---|---|
| Handshake `401` | Not claimed yet, or the token is wrong. Send or receive once, which claims, then watch. |
| Handshake `426` | The request was not a WebSocket upgrade, often a proxy that drops `Upgrade` headers. See the nginx settings in [Running the relay on Node](../relay/node.md#tls). |
| Closes after a fixed idle time | A proxy or load balancer idle timeout shorter than the relay's 25-second pings, or a proxy that does not pass WebSocket frames. Raise `proxy_read_timeout` or the idle timeout above 25 seconds. |
| Closes with `1000 erased` | The mailbox was erased. Treat the link as unpaired. |
| Closes with `1003` | The client sent a data frame. The watch is listen-only. |
| Never connects in a browser on Node 20 | No global `WebSocket`; pass one (see the [JavaScript SDK](../sdk/javascript.md#watch)). |

## Platform notes

**.NET: `PlatformNotSupportedException` or every envelope fails.** ChaCha20-Poly1305 is missing on
the system. Check `Envelope.IsSupported`; on Windows it needs 10 20H2 or newer, on Linux OpenSSL
1.1 or newer.

**.NET: credentials do not match the other device.** `Guid.ToByteArray()` returns the first three
fields little-endian. Pass the `Guid` itself to `Credentials`, or `ToByteArray(bigEndian: true)`.
See [UUID byte order](../protocol/keys.md#uuid-byte-order).

**JavaScript: `TypeError: Do not know how to serialize a BigInt`.** Sequence numbers are `BigInt`.
Convert with `toString()` before `JSON.stringify`.

**Android: `Cleartext HTTP traffic not permitted`.** Expected for an `http://` relay. Use HTTPS, or
for a local relay allow cleartext to that host in a debug-only network security config.

**iOS: requests fail in the background.** iOS suspends apps shortly after they leave the
foreground. Pull when the app becomes active and from background tasks, not from a watch.

**Any platform: requests hang without a network.** The Swift and Android SDKs time out after 30
seconds, .NET after `HttpClient`'s default of 100 seconds, and JavaScript only if you pass a `fetch`
with a timeout. Show the offline state from your own reachability check instead of waiting.

## Getting help

Open an issue in the repository of the component you use. Include the SDK and version, the
platform, the status codes you see, and what you expected. Never include a link key or a bearer
token; the mailbox name is safe to share.
