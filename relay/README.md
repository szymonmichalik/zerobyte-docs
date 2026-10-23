# Running a relay

The relay is [zerobyte-relay](https://github.com/szymonmichalik/zerobyte-relay). It is a blind,
quota-bound mailbox: it files opaque envelopes by app, mailbox, direction and sequence, and hands
them back. It cannot open them.

## Architecture

```
src/core.js          the whole protocol: routing, auth, claim, store, fetch, prune, erase, watch
src/apps.js          RELAY_APPS parsing and app id generation
src/node/            self-hosted adapter
    server.js        node:http server, one request at a time per mailbox, idle sweep
    storage.js       one SQLite file via node:sqlite
    websocket.js     minimal RFC 6455 server for the watch
src/cloudflare.js    Cloudflare adapter: Worker + one Durable Object per mailbox
bin/zerobyte-relay.js   command line entry point
```

`core.js` knows nothing about where it runs. Each adapter gives it three things: storage scoped to
one mailbox, a way to upgrade a request to a WebSocket, and the list of sockets already watching.
Both adapters run the same protocol code and pass the same tests.

The relay has **no runtime dependencies**. On Node it uses only built-in modules. `wrangler` is a
development dependency for the Cloudflare deployment.

## Choosing a deployment

| | [Node](node.md) | [Docker](docker.md) | [Cloudflare Workers](cloudflare-workers.md) |
|---|---|---|---|
| Runs on | Any machine with Node 22.13+ | Any machine with Docker | Cloudflare's network |
| HTTPS | You provide (reverse proxy) | Automatic (Caddy) | Automatic |
| Storage | One SQLite file | SQLite file in a volume | One SQLite Durable Object per mailbox |
| Scales | One process | One process | Per mailbox, automatically |
| Idle watchers | One TCP connection each | One TCP connection each | Hibernate, cost nothing |
| Best for | Development, small private deployments | A single server with a domain | Public apps, many users, global reach |

All three serve the same API at `https://<host>/v1/`.

## Steps common to every deployment

1. **Generate an app id** for each app that will use the relay:

   ```sh
   git clone https://github.com/szymonmichalik/zerobyte-relay.git
   cd zerobyte-relay
   npm run app-id
   ```

2. **Register it** in `RELAY_APPS`. See [Configuration](configuration.md).
3. **Deploy** with one of the guides.
4. **Point the SDKs at it** with the base URL `https://<host>/v1/`:

   | SDK | Code |
   |---|---|
   | Swift | `Relay(appID: appID, credentials: credentials, base: URL(string: "https://relay.example.com/v1/")!)` |
   | Kotlin | `Relay(appID, credentials, "https://relay.example.com/v1/")` |
   | JavaScript | `new Relay({ appID, credentials, base: "https://relay.example.com/v1/" })` |
   | .NET | `new Relay(appID, credentials, new Uri("https://relay.example.com/v1/"))` |

   The base must end with `/v1/`, including the trailing slash. The SDK appends the app id.

5. **Check it** with the live-relay tests of any SDK, or the curl walk-through in
   [HTTP API](../protocol/http-api.md#a-full-exchange-with-curl).

## What the relay keeps

| Kept | For how long |
|---|---|
| SHA-256 of one bearer token per mailbox | Until the mailbox is erased |
| Envelopes not yet pruned | Until the receiver prunes them or the mailbox is erased |
| The last sequence per direction | Until the mailbox is erased |
| Total bytes stored | Until the mailbox is erased |
| Idle timer | 30 days after the last write or claim |

It never receives the link key or any encryption key. It writes no request log. The Node server
prints one line at start-up and nothing after. The Cloudflare template has Workers Logs and traces
turned off. See [Operations](operations.md) for the infrastructure around it, which may log on its
own.

## Licence

The relay is AGPL-3.0-only. If you run a **modified** relay as a network service, you must offer
the modified source to the people who use it, for example with a link in your app's legal notices.
Running it unmodified, or running apps that only talk to a relay, imposes nothing: the SDKs are
Apache-2.0.

A relay you run must not present itself as `zerobyte.cloud` or suggest it is operated by the
ZeroByte project.
