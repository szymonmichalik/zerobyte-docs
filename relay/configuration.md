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

### App profiles

After the label, options separated by `;` give an app its own limits and
[extensions](../protocol/extensions.md):

```sh
RELAY_APPS="019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d=chat;push=com.example.chat;signal;ice;envelopes=1024;idle=90"
```

| Option | Range | Default | Meaning |
|---|---|---|---|
| `push=<topic>` | | off | Push through a provider module; for APNs the topic is the bundle id, FCM ignores it |
| `signal` | | off | Signal WebSocket for direct connections |
| `ice` | | off | STUN/TURN credentials |
| `envelopes=N` | 16 to 4096 | 64 | Envelopes waiting per direction |
| `mailbox=MiB` | 1 to 256 | 32 | Bytes waiting per mailbox |
| `envelope=KiB` | 1 to 1024 | 1024 | Plaintext per envelope |
| `idle=days` | 1 to 365 | 30 | Erase after this long without a write or claim |

An entry without options keeps the defaults, so existing configurations behave as before. A bad
option, a repeated one, or an envelope larger than the mailbox stops the relay at start-up.

## Node settings

| Variable | Default | Meaning |
|---|---|---|
| `RELAY_DATA` | `./data/relay.sqlite` | SQLite file. Its directory is created if missing. |
| `PORT` | `8080` | TCP port |
| `HOST` | `0.0.0.0` | Interface to listen on |

The Docker image sets `RELAY_DATA=/data/relay.sqlite`.

## Push and ICE settings

Needed only for apps with `push` or `ice`. All of a group or none: a partial group stops the relay
at start-up.

| Variable | Secret | Meaning |
|---|---|---|
| `APNS_TEAM_ID` | | Apple team id, 10 characters |
| `APNS_KEY_ID` | | Id of the APNs auth key, 10 characters |
| `APNS_PRIVATE_KEY` | yes | The key's `.p8` file content (Node also takes `APNS_PRIVATE_KEY_FILE`, a path) |
| `FCM_SERVICE_ACCOUNT` | yes | A Firebase service account's JSON key (Node also takes `FCM_SERVICE_ACCOUNT_FILE`, a path) |
| `ICE_STUN_URLS` | | STUN URLs, comma-separated |
| `TURN_URLS` | | coturn URLs (`turn:` / `turns:`), comma-separated |
| `TURN_SECRET` | yes | coturn's `static-auth-secret` (`use-auth-secret` in `turnserver.conf`) |
| `CF_TURN_KEY_ID` | | Cloudflare Realtime TURN key id |
| `CF_TURN_API_TOKEN` | yes | That key's API token |

The `APNS_*` variables configure the APNs module (`src/apns/` in the relay). It is separate from
the relay core: without them it is not loaded. The same holds for FCM below; with neither, the
relay sends no pushes. Other push services
are added as modules of the same shape (see [Push](../protocol/extensions.md#push)).

One APNs key serves every app and both environments; the relay picks the host from each
registration's `environment`. The provider token is signed on the relay and reused for 40 minutes.
On Node, pushes go over HTTP/2 with `node:http2`. On Workers, `fetch` negotiates HTTP/2 with Apple
when deployed, but `wrangler dev` does not, so test real pushes on Node or a deployed relay.

`FCM_SERVICE_ACCOUNT` configures the FCM module (`src/fcm/`). Create the key in the Firebase
console under Project settings, Service accounts; it needs no role beyond what Firebase gives it
by default. The relay signs a short assertion with it, exchanges it for an access token that it
reuses for 50 minutes, and sends through the FCM HTTP v1 API with `fetch`, on Node and Workers
alike. One key serves every Android app in that Firebase project.

## Cloudflare settings

| `wrangler.toml` key | Value |
|---|---|
| `[vars] RELAY_APPS` | As above |
| `[vars]` | `APNS_TEAM_ID`, `APNS_KEY_ID`, `ICE_STUN_URLS`, `TURN_URLS`, `CF_TURN_KEY_ID` if used |
| Secrets | `npx wrangler secret put APNS_PRIVATE_KEY`, `FCM_SERVICE_ACCOUNT` (and `TURN_SECRET`, `CF_TURN_API_TOKEN`) |
| `[[durable_objects.bindings]]` | `name = "MAILBOX"`, `class_name = "Mailbox"`; do not change |
| `[[migrations]]` | `tag = "v1"`, `new_sqlite_classes = ["Mailbox"]`; do not change |
| `[observability] enabled` | `false` |
| `routes` | Optional custom domain |

## Default limits

The defaults are `DEFAULT_LIMITS` in `src/core.js`. An app without options gets exactly these; an
[app profile](#app-profiles) may change the first, third, fourth and last within the ranges above.
The fetch limit and the minimum envelope are fixed.

| Constant | Value |
|---|---|
| `envelopeBytes` | 1,048,614 |
| `minimumEnvelopeBytes` | 38 |
| `mailboxBytes` | 33,554,432 |
| `envelopesPerDirection` | 64 |
| `fetchLimit` | 16 |
| `idleMilliseconds` | 2,592,000,000 (30 days) |
