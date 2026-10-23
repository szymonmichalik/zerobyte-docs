# Relay configuration

## `RELAY_APPS`

The apps a relay serves. Required on every deployment.

```
RELAY_APPS="<app id>[=<label>][,<app id>[=<label>]...]"
```

- Each app id is a lowercase UUIDv7, for example `019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d`.
- Entries are separated by commas or whitespace.
- An entry may carry a label after `=`. Labels are for the operator's reference; the relay does not
  use them in responses.
- An invalid entry stops the relay at start-up with the message
  `RELAY_APPS: "<value>" is not a lowercase UUIDv7`.

Examples:

```sh
RELAY_APPS=019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d
RELAY_APPS="019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d=notes,019a0d4f-0a2b-7c3d-8e4f-5a6b7c8d9e0f=photos"
```

### Generating app ids

```sh
npm run app-id
# or
node bin/zerobyte-relay.js app-id
```

Each run prints a fresh UUIDv7 (RFC 9562 §5.7): 48 bits of the current Unix time in milliseconds,
then random bits, with the version and variant set. The id is not secret: it ships inside your app.
Generate one per app, not per user or per device.

### Why app ids exist

- A relay answers only for apps its operator registered, so it cannot be used as free storage by
  strangers who find its address.
- Mailboxes of different apps are kept apart even if two links derived the same mailbox id.
- An unregistered app and a typo get the same `404`, so nobody can enumerate which apps a relay
  serves.

The app id does not authorise anything by itself; the per-mailbox bearer token does.

## Node settings

| Variable | Default | Meaning |
|---|---|---|
| `RELAY_DATA` | `./data/relay.sqlite` | SQLite file. Its directory is created if missing. |
| `PORT` | `8080` | TCP port |
| `HOST` | `0.0.0.0` | Interface to listen on |

The Docker image sets `RELAY_DATA=/data/relay.sqlite`.

## Cloudflare settings

| `wrangler.toml` key | Value |
|---|---|
| `[vars] RELAY_APPS` | As above |
| `[[durable_objects.bindings]]` | `name = "MAILBOX"`, `class_name = "Mailbox"`; do not change |
| `[[migrations]]` | `tag = "v1"`, `new_sqlite_classes = ["Mailbox"]`; do not change |
| `[observability] enabled` | `false` |
| `routes` | Optional custom domain |

## Fixed limits

The protocol limits are constants in `src/core.js`, not settings. Clients rely on them; a relay
with different limits would not be a Relay Protocol v1 relay.

| Constant | Value |
|---|---|
| `envelopeBytes` | 1,048,614 |
| `minimumEnvelopeBytes` | 38 |
| `mailboxBytes` | 33,554,432 |
| `envelopesPerDirection` | 64 |
| `fetchLimit` | 16 |
| `idleMilliseconds` | 2,592,000,000 (30 days) |
