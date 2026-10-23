# HTTP API

## Base path

```
https://<relay host>/v1/<app id>/mailbox/<mailbox name>
```

- `<app id>` is a registered UUIDv7 in lowercase:
  `^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`.
- `<mailbox name>` is the mailbox id in lowercase hex: `^[0-9a-f]{32}$`.
- `<dir>` below is `p2c` or `c2p`.
- `<seq>` is a decimal sequence number from 1 to 2^64 - 1, with no leading zeros.

An unknown app id, a malformed mailbox name, the bare host and every path outside this tree answer
the same `404`. A client cannot tell an unregistered app from a typo, and a scanner learns nothing
about which apps a relay serves.

## Authentication

Every request carries:

```
Authorization: Bearer <bearer>
```

where `<bearer>` is the 43-character base64url token from [Keys and derivation](keys.md). A
missing or malformed bearer gets `401` before the mailbox is consulted. On a WebSocket upgrade, a
browser may send the bearer as a subprotocol instead; see [Watch](watch.md).

The token is never a cookie, and the relay sets no cookies.

## Endpoints

| Request | Does | Success | Other answers |
|---|---|---|---|
| `PUT /mailbox/<id>` | Claim the mailbox | `201` claimed, `204` already yours | `400`, `403` |
| `DELETE /mailbox/<id>` | Erase the mailbox | `204` | `401` |
| `PUT /mailbox/<id>/<dir>/<seq>` | Store one envelope | `201` | `400`, `401`, `409`, `413`, `507` |
| `GET /mailbox/<id>/<dir>?after=&limit=` | Fetch envelopes | `200` | `400`, `401` |
| `HEAD /mailbox/<id>/<dir>` | Last stored sequence | `200` | `401` |
| `DELETE /mailbox/<id>/<dir>?through=` | Prune | `204` | `400`, `401` |
| `GET /mailbox/<id>/<dir>/watch` | WebSocket | `101` | `401`, `426` |
| `OPTIONS` any of the above | CORS preflight | `204` | `404` |

Paths in the table are relative to `/v1/<app id>`.

### Claim

```
PUT /v1/<app>/mailbox/<id>
Authorization: Bearer <bearer>
Content-Type: application/json

{"tokenDigest": "<token digest>"}
```

The first bearer to claim a mailbox owns it. The body must name the SHA-256 of the bearer being
presented, as lowercase hex. This guards against a client with a bug locking a mailbox under a
token it cannot reproduce: if the digest does not match the bearer, the relay answers `400` and
claims nothing.

| Answer | Meaning |
|---|---|
| `201` | The mailbox was free and is now yours |
| `204` | It was already claimed with this token; nothing changed except the idle timer |
| `403` | It belongs to another token |
| `400` | The body is not JSON or names a different digest |

Claiming is idempotent. Both devices of a link may claim; the second one gets `204`. A claim
restarts the 30-day idle timer.

Every other request to a mailbox that has not been claimed answers `401`.

### Erase

```
DELETE /v1/<app>/mailbox/<id>
```

Deletes both directions, the stored digest and the idle timer, and closes every watcher with close
code `1000` and reason `erased`. Answers `204`. The next claim, by any token, starts afresh.

### Store an envelope

```
PUT /v1/<app>/mailbox/<id>/<dir>/<seq>
Content-Type: application/octet-stream

<envelope bytes>
```

The relay checks, without opening the envelope:

| Check | Failure |
|---|---|
| Body at most 1 MiB + 38 bytes (declared `Content-Length` or actual length) | `413` |
| Body at least 38 bytes, version byte `1` | `400` |
| Direction byte matches `<dir>` | `400` |
| Sequence in the header equals `<seq>` | `400` |
| `<seq>` is above the last sequence ever stored in this direction | `409` with `X-Relay-Last: <last>` |
| Fewer than 64 envelopes waiting in this direction | `507` |
| Mailbox total stays within 32 MiB | `507` |

On success it stores the envelope, records `<seq>` as the direction's last sequence, restarts the
idle timer, notifies watchers of the direction and answers `201`.

A `<seq>` that is not a valid sequence number answers `404`. Any method other than `PUT` on this
path answers `405` with `Allow: PUT`.

### Fetch

```
GET /v1/<app>/mailbox/<id>/<dir>?after=<seq>&limit=<n>
```

| Parameter | Default | Rule |
|---|---|---|
| `after` | `0` | `0` or a valid sequence. Returns envelopes with a higher sequence. |
| `limit` | `16` | A positive integer. Values above 16 are treated as 16. |

Answers `200` with `Content-Type: application/octet-stream` and the [wire body](#wire-body) holding
up to `limit` envelopes, oldest first.

| Header | Meaning |
|---|---|
| `X-Relay-More` | `1` if more envelopes are waiting after the last one returned, else `0` |
| `X-Relay-Last` | The last sequence ever stored in this direction; absent if none |

An empty result is a `200` with the body `00000000`.

### Last sequence

```
HEAD /v1/<app>/mailbox/<id>/<dir>
```

Answers `200` with `X-Relay-Last: <seq>`, the highest sequence ever stored in the direction, or no
such header if nothing was ever stored. The value survives pruning, so a sender that lost its state
can continue from it.

### Prune

```
DELETE /v1/<app>/mailbox/<id>/<dir>?through=<seq>
```

Deletes every envelope in the direction with a sequence up to and including `through`. Answers
`204`, also when there was nothing to delete. `through` is required; a missing or invalid value
answers `400`. Pruning does not change `X-Relay-Last` and does not restart the idle timer.

### Watch

`GET /v1/<app>/mailbox/<id>/<dir>/watch` upgrades to a WebSocket. A request without
`Upgrade: websocket` answers `426`. See [Watch WebSocket](watch.md).

### Other methods

On the mailbox path, methods other than `PUT` and `DELETE` answer `405` with `Allow: PUT, DELETE`.
On a direction path, methods other than `GET`, `HEAD` and `DELETE` answer `405` with
`Allow: GET, HEAD, DELETE`. A direction other than `p2c` or `c2p` answers `404`.

## Wire body

A fetch response packs envelopes back to back:

```
count (UInt32 BE)                          at most 64
repeat count times:
    length (UInt32 BE) ‖ envelope (length bytes)
```

A client must reject a body that:

- is shorter than 4 bytes;
- has a count above 64;
- has an envelope length above 1 MiB + 38 or beyond the end of the body;
- has bytes left over after the last envelope.

| Envelopes | Body |
|---|---|
| none | `00000000` |
| one of 38 bytes | `00000001` `00000026` followed by 38 bytes |

The relay never returns more than 16 envelopes; the limit of 64 is what a client accepts.

## Errors

Every error has a small JSON body. Clients act on the status code alone.

```json
{"error": "sequence_conflict", "status": 409}
```

| Status | `error` | Typical cause |
|---|---|---|
| `400` | `bad_request` | Envelope header does not match the path; bad query parameter; bad claim body |
| `401` | `unauthorized` | Missing or malformed bearer; mailbox not claimed; wrong token |
| `403` | `forbidden` | Claim of a mailbox owned by another token |
| `404` | `not_found` | Unknown app, malformed path, unknown direction |
| `405` | `method_not_allowed` | Wrong method for the path; see `Allow` |
| `409` | `sequence_conflict` | Sequence not above the last; see `X-Relay-Last` |
| `413` | `envelope_too_large` | Envelope over 1 MiB + 38 bytes |
| `426` | `upgrade_required` | Watch path without a WebSocket upgrade |
| `507` | `mailbox_full` | 64 envelopes waiting in the direction, or 32 MiB in the mailbox |

The SDKs fold these into a few codes:

| Status | SDK code |
|---|---|
| `401`, `403` | `unauthorized` |
| `409` | `sequence_conflict`, with the relay's last sequence |
| `413`, `507` | `quota_exceeded` |
| anything else | `unavailable`, with the status |
| unparseable body | `malformed_response` |

## Response headers

Every response carries:

```
Cache-Control: no-store
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: X-Relay-Last, X-Relay-More
```

A preflight `OPTIONS` on any valid path also gets:

```
Access-Control-Allow-Methods: GET, HEAD, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

Allowing every origin is safe because authority comes only from the bearer header. A page that
does not hold the token gets `401`.

## Concurrency

The relay handles requests to one mailbox one at a time, in arrival order. Two writes racing for
the same sequence resolve cleanly: one gets `201`, the other `409`. Different mailboxes do not
wait for each other.

## A full exchange with curl

The bearer and digest below come from credential set 0 of the [test vectors](test-vectors.md).
The envelope is that set's first envelope. Set `APP` to an app id registered on your relay.

```sh
BASE=http://127.0.0.1:8080/v1/$APP/mailbox/41fa13c183c0c2092a085390c172cba1
AUTH="Authorization: Bearer 16Lm3t_PFz5tqDn_w8cIkbkUtj8zrxVcMgO4_qSNX74"

# Claim
curl -i -X PUT "$BASE" -H "$AUTH" -H 'Content-Type: application/json' \
  -d '{"tokenDigest":"8da1975827ae6db05a1f3be74348c315975316e3e10c22b12abfa2febc7d8b51"}'

# Store envelope 1 (hex from vectors.json, envelopes[0].envelope)
echo -n "<envelope hex>" | xxd -r -p > env1.bin
curl -i -X PUT "$BASE/p2c/1" -H "$AUTH" -H 'Content-Type: application/octet-stream' \
  --data-binary @env1.bin

# Last sequence
curl -I "$BASE/p2c" -H "$AUTH"

# Fetch and show the wire body
curl -s "$BASE/p2c?after=0&limit=16" -H "$AUTH" | xxd | head

# Prune and erase
curl -i -X DELETE "$BASE/p2c?through=1" -H "$AUTH"
curl -i -X DELETE "$BASE" -H "$AUTH"
```
