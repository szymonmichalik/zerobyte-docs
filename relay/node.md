# Running the relay on Node

## Requirements

- Node 22.13 or newer. The relay uses the built-in `node:sqlite` module.
- A writable directory for the database.
- A TLS-terminating reverse proxy for anything other than local development.

No `npm install` is needed: the relay has no runtime dependencies.

## Start

```sh
git clone https://github.com/szymonmichalik/zerobyte-relay.git
cd zerobyte-relay
npm run app-id                                     # once per app
RELAY_APPS=<app id> RELAY_DATA=./data/relay.sqlite PORT=8080 npm start
```

Or run the entry point directly:

```sh
RELAY_APPS=<app id> node bin/zerobyte-relay.js
```

On success it prints exactly one line:

```
ZeroByte Relay v1 on 0.0.0.0:8080, 1 app(s), data in ./data/relay.sqlite
```

If `RELAY_APPS` is empty or contains something that is not a lowercase UUIDv7, it prints the reason
and exits with status 64. The directory of `RELAY_DATA` is created if missing.

`SIGINT` and `SIGTERM` close every watch socket with `1001 going away`, close the database and
exit with status 0.

## Settings

| Variable | Default | Meaning |
|---|---|---|
| `RELAY_APPS` | (required) | Registered app ids, comma-separated, each optionally `=label` |
| `RELAY_DATA` | `./data/relay.sqlite` | SQLite database file |
| `PORT` | `8080` | TCP port |
| `HOST` | `0.0.0.0` | Interface to listen on. Use `127.0.0.1` behind a local reverse proxy. |

See [Configuration](configuration.md) for the format of `RELAY_APPS`.

## TLS

The server speaks plain HTTP. Never expose it directly to the internet: bearer tokens would travel
in clear text. Put a reverse proxy in front.

### Caddy

```
relay.example.com {
        reverse_proxy 127.0.0.1:8080
}
```

Caddy obtains and renews a certificate and passes WebSocket upgrades through. It writes no access
log unless you add a `log` directive.

### nginx

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name relay.example.com;

    ssl_certificate     /etc/letsencrypt/live/relay.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/relay.example.com/privkey.pem;

    access_log off;
    client_max_body_size 2m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 1h;
    }
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

`client_max_body_size` must be above 1 MiB + 38 bytes, or nginx answers `413` before the relay
sees the envelope. `proxy_read_timeout` keeps idle watch sockets open; the relay pings every 25
seconds, so anything above that works.

## Running as a service

A systemd unit:

```ini
# /etc/systemd/system/zerobyte-relay.service
[Unit]
Description=ZeroByte Relay
After=network.target

[Service]
Type=simple
User=zerobyte
WorkingDirectory=/opt/zerobyte-relay
Environment=RELAY_APPS=<app id>
Environment=RELAY_DATA=/var/lib/zerobyte-relay/relay.sqlite
Environment=HOST=127.0.0.1
Environment=PORT=8080
ExecStart=/usr/bin/node bin/zerobyte-relay.js
Restart=on-failure
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/zerobyte-relay
StateDirectory=zerobyte-relay

[Install]
WantedBy=multi-user.target
```

```sh
sudo useradd --system --home /opt/zerobyte-relay zerobyte
sudo git clone https://github.com/szymonmichalik/zerobyte-relay.git /opt/zerobyte-relay
sudo systemctl daemon-reload
sudo systemctl enable --now zerobyte-relay
```

systemd's journal records the start-up line and nothing else, because the relay prints nothing
else.

## How the Node adapter works

- **One request at a time per mailbox.** Requests to the same mailbox are queued and run in order,
  the way a Durable Object runs them. Different mailboxes run concurrently.
- **Storage.** One SQLite file in WAL mode with `synchronous = NORMAL` and `secure_delete = ON`.
  Secure delete overwrites pruned ciphertext with zeros, so read envelopes do not linger in free
  pages of the file.
- **Idle expiry.** Every 60 seconds the server erases up to 256 mailboxes whose 30-day timer has
  passed, repeating on the next sweep if more are due.
- **WebSocket.** A minimal RFC 6455 server: handshake, text frames out, ping every 25 seconds,
  answers to ping and close. Any data frame from a client closes the socket.
- **Body size.** The server stops reading a request body just past the envelope limit, so an
  oversized upload gets `413` without being buffered whole.

## Embedding

`src/node/server.js` exports `createRelayServer(options)`, which returns a `node:http` server, and
`createRelay(options)`, which returns an object with a `fetch(request)` method for use in tests or
another server.

```js
// The package is not on npm; import from a checkout or a git dependency.
import { createRelayServer } from "./zerobyte-relay/src/node/server.js";

const server = createRelayServer({ apps: "<app id>", database: "./data/relay.sqlite" });
server.listen(8080, "127.0.0.1");
```

`database: ":memory:"` keeps everything in memory, which is useful in tests.
