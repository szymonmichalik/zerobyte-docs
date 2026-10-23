# Running the relay on Cloudflare Workers

On Cloudflare the relay is one Worker and one SQLite-backed Durable Object per mailbox. Each
mailbox runs its requests one at a time, close to the devices that use it, and idle watch sockets
hibernate at no cost.

## Requirements

- A Cloudflare account. Durable Objects with SQLite storage are available on the Workers Free and
  Paid plans; check Cloudflare's current limits for your expected traffic.
- Node 22 or newer and npm, to run `wrangler`.

## Deploy

```sh
git clone https://github.com/szymonmichalik/zerobyte-relay.git
cd zerobyte-relay
npm install                        # installs wrangler, a development dependency
npm run app-id
```

Edit `wrangler.toml`:

```toml
name = "my-zerobyte-relay"
main = "src/cloudflare.js"
compatibility_date = "2026-10-01"

# routes = [{ pattern = "relay.example.com", custom_domain = true }]

[vars]
RELAY_APPS = "<app id>"

[[durable_objects.bindings]]
name = "MAILBOX"
class_name = "Mailbox"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["Mailbox"]

[observability]
enabled = false
```

| Key | What to change |
|---|---|
| `name` | Your Worker's name; it becomes `<name>.<account>.workers.dev` |
| `routes` | Uncomment and set your domain to serve the relay on it. The domain must be a zone in the same Cloudflare account. |
| `RELAY_APPS` | Your app ids, comma-separated |
| `observability` | Keep `enabled = false` so Cloudflare keeps no request logs or traces |

Leave the Durable Object binding and the migration as they are: the code expects the binding
`MAILBOX` and the class `Mailbox`.

Log in and deploy:

```sh
npx wrangler login
npx wrangler deploy
```

Your SDK base URL is `https://relay.example.com/v1/` with a custom domain, or
`https://my-zerobyte-relay.<account>.workers.dev/v1/` without one.

## Local development

```sh
npx wrangler dev
```

This runs the Worker and its Durable Objects locally on `http://127.0.0.1:8787`. Use
`http://127.0.0.1:8787/v1/` as the base URL. The Node adapter is usually quicker for local work and
behaves the same.

## Changing the registered apps

`RELAY_APPS` is a plain variable. Edit it in `wrangler.toml` and deploy again; existing mailboxes
are untouched. Removing an app id makes its mailboxes unreachable (`404`) but does not delete them;
they expire 30 days after their last write or claim.

## How the Cloudflare adapter works

- The Worker parses the path. Anything that is not a valid request for a registered app gets `404`
  without touching a Durable Object.
- Each mailbox is the Durable Object named `<app id>/<mailbox id>`, so two apps never share one.
- The object stores its state in its own SQLite storage and uses a Durable Object alarm for the
  30-day idle expiry.
- Watch sockets use the WebSocket Hibernation API: the object can be evicted from memory while
  sockets stay open, and is woken by the next store, which notifies them.

## Logging and analytics

With `observability.enabled = false`, the Worker writes no logs and no traces. The relay code
binds no Analytics Engine dataset in this template, so it counts nothing. Cloudflare still sees
the traffic at its edge as it does for any Worker; see Cloudflare's privacy documentation for what
the platform itself retains.
