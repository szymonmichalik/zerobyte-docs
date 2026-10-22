# Operating a relay

## TLS is mandatory

The bearer token travels in every request. Without TLS anyone on the path can copy it and then
read the (still encrypted) envelopes, fill the mailbox, prune envelopes before the receiver gets
them or erase the mailbox. They still cannot read or forge messages, but they can deny service.
Serve the relay only over HTTPS; Docker with Caddy and Cloudflare do this for you.

The SDKs accept `http://` base URLs for local development. Do not ship an app with one.

## Logging

The relay itself logs nothing per request. Keep it that way around it:

| Component | Setting |
|---|---|
| Caddy | No `log` directive (the default) |
| nginx | `access_log off;` in the relay's `server` block |
| Load balancers, CDNs | Disable access logs or set short retention |
| Cloudflare Workers | `observability.enabled = false` |

Error logs of the proxy may still record client addresses on failures. Rotate them with short
retention.

## Storage

### Node

One SQLite file, by default `./data/relay.sqlite`, with WAL files next to it (`-wal`, `-shm`).

- Size: at most 32 MiB per active mailbox, usually far less, since receivers prune what they read.
- Pruned and expired data is overwritten with zeros (`secure_delete`), then the space is reused.
  The file does not shrink on its own. To reclaim space, stop the relay and run
  `sqlite3 relay.sqlite 'VACUUM;'`.
- Put the file on a local disk. SQLite over a network file system is unreliable.

### Backups

The relay holds only transient, end-to-end encrypted data that is deleted once read. Backing it up
is usually unnecessary and works against the design: a backup keeps envelopes the relay was meant
to forget. If you lose the database, devices re-claim their mailboxes on the next request; whatever
was waiting is lost and senders continue from sequence 1. See
[Link lifecycle](../guides/link-lifecycle.md#the-mailbox-was-reset).

### Cloudflare

Each mailbox is a Durable Object with its own storage, managed by Cloudflare. There is nothing to
size or vacuum.

## Monitoring

The relay has no metrics endpoint. Useful checks:

- **Liveness**: `GET https://relay.example.com/` answers `404` with
  `{"error":"not_found","status":404}`. Any other answer, or none, means the relay is down.
- **End to end**: run the live-relay tests of an SDK against it on a schedule, with a dedicated
  app id. They claim, write, fetch, prune and erase a fresh mailbox.
- **Host**: disk space for the SQLite file, memory, open file descriptors (one per watch socket).

## Capacity

A single Node process handles a large number of small mailboxes. The main costs are:

- one TCP connection per open watch socket;
- disk writes for each stored envelope;
- memory for request bodies in flight, at most about 1 MiB each.

Raise the file descriptor limit (`LimitNOFILE` in systemd) if many devices keep a watch open. For
many thousands of concurrent watchers, prefer the Cloudflare deployment, where idle sockets
hibernate.

## Upgrading

- **Node**: `git pull`, then restart. The SQLite schema is created with `IF NOT EXISTS` and is
  stable within v1.
- **Docker**: `git pull && docker compose up -d --build` in `deploy/`.
- **Cloudflare**: `git pull`, keep your `wrangler.toml` changes, `npx wrangler deploy`.

Watch sockets drop during a restart; clients reconnect and fetch. Requests in flight may fail with
a network error; clients retry.

## Moving a relay

Devices find the relay by the base URL compiled into your app. To move:

1. Deploy the new relay with the same `RELAY_APPS`.
2. Ship an app version that uses the new base URL.
3. Keep the old relay running until the old app versions are gone, or until every link has had a
   chance to drain.

Mailboxes do not need to be migrated: on the new relay, the first request re-claims each mailbox.
Envelopes still waiting on the old relay are lost unless both relays run in parallel while
clients drain the old one.

## Abuse

A relay serves only registered app ids, and only clients holding a mailbox's token can use that
mailbox. Quotas bound every mailbox at 32 MiB and 64 envelopes per direction. Someone who
extracts your app id from your app can still create mailboxes with tokens of their own; rate
limiting in front of the relay (by IP, at the proxy or with Cloudflare rules) bounds that.
