# Watch WebSocket

Polling a mailbox wastes battery and requests. A device that is in the foreground instead opens a
WebSocket per direction it reads and fetches when the relay says something arrived.

```
GET /v1/<app>/mailbox/<id>/<dir>/watch
Upgrade: websocket
Sec-WebSocket-Protocol: zerobyte.v1
Authorization: Bearer <bearer>
```

## Messages

The relay sends one **text** message per envelope stored in the direction after the socket opened.
The message is the envelope's sequence number in decimal, for example `17`. Nothing else is ever
sent: no envelope bytes, no sizes.

The client still fetches, opens and prunes as usual. The notification tells the client only what
it would learn by fetching, so the relay learns nothing new from it. Treat each message as "fetch
now"; you do not need the number itself.

The watch is listen-only. A data frame from the client closes the socket with code `1003`
(`listen only`). Ping and pong are handled by the WebSocket layer; the Node relay pings every 25
seconds.

## Subprotocols

| Subprotocol | Purpose |
|---|---|
| `zerobyte.v1` | Optional. If offered, the relay answers with it in `Sec-WebSocket-Protocol`. |
| `zerobyte.bearer.<bearer>` | The bearer, for clients that cannot set headers. Never echoed back. |

Browsers cannot set an `Authorization` header on a WebSocket. A browser client therefore offers
both subprotocols:

```js
new WebSocket(url, ["zerobyte.v1", `zerobyte.bearer.${bearer}`]);
```

The relay accepts the bearer from a subprotocol only on a WebSocket upgrade, never on a plain HTTP
request. It answers with `zerobyte.v1` only, so the token does not appear in the response.

Native clients send `Authorization: Bearer` and offer `zerobyte.v1`.

## When the socket closes

| Close | Meaning | What to do |
|---|---|---|
| `1000 erased` | The mailbox was erased, usually because the other device unpaired | Stop. Treat the link as unpaired. |
| `1001 going away` | The relay is shutting down or restarting | Reconnect with backoff |
| `1003 listen only` | The client sent a data frame | Fix the client |
| `1006`, network error | Connection lost | Reconnect with backoff |
| Handshake answered `401` | The mailbox is not claimed or the token is wrong | Claim first, or treat the link as unpaired |

## The rule that makes it reliable

Only envelopes stored after the socket opened are announced. Anything stored while you were
disconnected is announced by nobody. So:

1. Open the socket.
2. When it opens, fetch once.
3. On every message, fetch.
4. On close, reconnect with exponential backoff and jitter, then go back to step 2.

A fetch that runs while another is in progress is harmless with the SDKs: `Link` runs one
operation at a time, and the second `receive` finds nothing new or picks up what arrived meanwhile.

## On mobile

iOS and Android suspend sockets in the background. Close the watch when your app goes to the
background and reopen it when it comes back; the fetch on open catches up. To wake a suspended app
when data arrives, you need a push channel, which ZeroByte does not provide. Many apps simply
fetch on launch, on foreground and on a background refresh task.

## Costs

On Cloudflare Workers an idle watcher hibernates, so an open socket costs nothing while no envelope
arrives. On Node each watcher holds one TCP connection.
