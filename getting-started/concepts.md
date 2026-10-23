# Concepts

## Link

A **link** joins exactly two devices of one app. It is defined by two values both devices hold:

| Value | Size | Secret | Purpose |
|---|---|---|---|
| Link key | 32 bytes | Yes | The root secret. Every key and identifier is derived from it. |
| Link id | 16 bytes | No | Names the link. Used as the HKDF salt, so two links never share a mailbox even if a key repeated. A UUID works. |

The link key must come from a cryptographically secure random generator or a key agreement. It
never leaves the two devices. The link id may be any 16 bytes the devices agree on; a random
UUID (version 4) is the simplest choice.

## Primary and companion

The two ends of a link have roles:

- The **primary** holds the source of truth, usually the phone.
- The **companion** mirrors it: a laptop, tablet, watch or browser.

The protocol only uses the roles to name the two directions. What they mean in your app is up to
you, but each device must know which role it plays, and the two devices must play different roles.

## Directions

Each link has two one-way channels:

| Direction | Wire value | Path segment | Written by | Read by |
|---|---|---|---|---|
| Primary to companion | `1` | `p2c` | Primary | Companion |
| Companion to primary | `2` | `c2p` | Companion | Primary |

Each direction has its own encryption key and its own sequence numbers. Neither side can replay
the other's envelopes back to it, because an envelope sealed for `p2c` will not open as `c2p`.

In the SDKs you choose the direction a `Link` **sends**; it reads the reverse.

| SDK | Primary | Companion |
|---|---|---|
| Swift | `.primaryToCompanion` | `.companionToPrimary` |
| Kotlin | `Direction.P2C` | `Direction.C2P` |
| JavaScript | `Direction.P2C` | `Direction.C2P` |
| .NET | `Direction.P2C` | `Direction.C2P` |

## Mailbox

The **mailbox** is where a link's envelopes wait on the relay. Its id is 16 bytes derived from the
link key, written in URLs as 32 lowercase hex characters. It holds both directions.

The first device to present a valid bearer token **claims** the mailbox. From then on, only that
token can use it. Both devices derive the same token, so both can use the mailbox; anyone else
gets `401` or `403`.

A mailbox holds at most 32 MiB and 64 unread envelopes per direction. It is erased when a device
unpairs, or automatically 30 days after the last write or claim.

## Bearer token

A 32-byte value derived from the link key, sent as `Authorization: Bearer <token>` in unpadded
base64url (43 characters). The relay stores only its SHA-256, so a copy of the relay's storage does
not let anyone use a mailbox.

## App id

Every relay request path starts with an **app id**: a UUIDv7 that the relay operator registers.
It is not a secret; it ships inside your app. It keeps unknown clients off the relay and keeps
each app's mailboxes apart: the same mailbox id under two app ids is two different mailboxes.

You generate an app id once per app with `npm run app-id` in the relay repository, then register it
in the relay's `RELAY_APPS` setting. See [Configuration](../relay/configuration.md).

## Envelope

One sealed message:

```
version (1) ‖ direction (1) ‖ sequence (8) ‖ nonce (12) ‖ ciphertext (n) ‖ tag (16)
```

The first 10 bytes are the clear header. The relay checks that the header matches the URL it was
stored under, but it cannot open the envelope. Plaintext is at most 1 MiB; overhead is 38 bytes.
See [Envelope format](../protocol/envelope.md).

## Sequence number

A 64-bit unsigned number, starting at 1 and strictly increasing within one direction. The sender
picks the next number; the relay refuses a number that is not above the last one stored (`409`).
The receiver refuses a number at or below the last one it accepted, which defeats replays.

Persist the receiver's **last received** sequence between launches. A receiver that forgets it
would accept an old envelope again. The SDKs expose it as `lastReceived` and accept it back when
you construct a `Link`.

## Watch

A WebSocket per direction that announces new sequence numbers as they are stored. It carries no
data. A device that gets a notification fetches as usual. See [Watch WebSocket](../protocol/watch.md).

## Prune

After opening what it fetched, the receiver tells the relay to drop everything up to the highest
sequence it saw. The relay keeps nothing it does not have to.
