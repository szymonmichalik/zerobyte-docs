# Running the relay with Docker

The repository includes a `Dockerfile` and, in `deploy/`, a Compose file that puts Caddy in front
for automatic HTTPS.

## With automatic HTTPS

Requirements: a server with Docker and Compose, ports 80 and 443 reachable from the internet, and
a DNS record for your relay's name pointing at the server.

```sh
git clone https://github.com/szymonmichalik/zerobyte-relay.git
cd zerobyte-relay
npm run app-id                     # without Node on the host, see the end of this page
cd deploy
RELAY_DOMAIN=relay.example.com RELAY_APPS=<app id> docker compose up -d
```

Compose builds the relay image from the repository, starts it and starts Caddy. Caddy obtains a
certificate for `RELAY_DOMAIN` on first start and renews it. Both containers restart unless stopped.

| Volume | Holds |
|---|---|
| `relay-data` | The SQLite database at `/data/relay.sqlite` |
| `caddy-data` | Certificates and Caddy state |

The `Caddyfile` has no `log` directive, so Caddy keeps no access log.

To keep the settings out of your shell history, put them in `deploy/.env`:

```sh
RELAY_DOMAIN=relay.example.com
RELAY_APPS=<app id>,<another app id>
```

Then `docker compose up -d` without the variables.

### Updating

```sh
cd zerobyte-relay
git pull
cd deploy
docker compose up -d --build
```

Watchers are disconnected during the restart and reconnect on their own.

## The image alone

```sh
docker build -t zerobyte-relay .
docker run -d --name zerobyte-relay --restart unless-stopped \
  -p 127.0.0.1:8080:8080 \
  -v relay-data:/data \
  -e RELAY_APPS=<app id> \
  zerobyte-relay
```

Put your own TLS proxy in front of port 8080; see [Node](node.md#tls).

### Image details

| | |
|---|---|
| Base | `node:24-alpine` |
| User | `node` (not root) |
| Working directory | `/app` |
| Volume | `/data` |
| Port | `8080` |
| Defaults | `PORT=8080`, `HOST=0.0.0.0`, `RELAY_DATA=/data/relay.sqlite`, `NODE_ENV=production` |
| Command | `node bin/zerobyte-relay.js` |

The image copies only `package.json`, `LICENSE`, `NOTICE`, `bin/` and `src/`.

## Generating an app id without Node on the host

```sh
docker run --rm -v "$PWD":/app -w /app node:24-alpine node bin/zerobyte-relay.js app-id
```
