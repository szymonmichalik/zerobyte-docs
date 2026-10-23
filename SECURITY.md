# Security

ZeroByte moves end-to-end encrypted messages between two devices through a relay that cannot read
them. This page states what that protects against, what it does not, and how to report a
vulnerability.

## What it protects against

| Threat | Protection |
|---|---|
| **The relay operator**, whoever runs the relay | The relay never receives a key. It cannot read, alter or forge an envelope without the receiver noticing. |
| **Someone with a copy of the relay's database** | The database holds ciphertext, the SHA-256 of each mailbox's token, and sequence numbers. No keys, no plaintext. Pruned and expired data is overwritten. |
| **Moved or replayed envelopes** | The header and mailbox id are authenticated. An envelope cannot be moved to another mailbox, another direction or another sequence, and receivers refuse any sequence they have already passed. |
| **Other apps and other mailboxes** | The mailbox id and the access token are derived from the link key. Mailboxes are filed per app id. A token opens only its own mailbox. |
| **A network observer** | TLS between each device and the relay, and end-to-end encryption inside it. |

## What it does not protect against

| Limitation | What it means for you |
|---|---|
| **Metadata** | The relay sees when and how much each mailbox exchanges, and the IP address of each request while it lasts. It does not log them, but it sees them. Pad and schedule messages if timing or size reveals something. See [Designing messages](guides/message-design.md#what-not-to-put-in-messages). |
| **A compromised device** | Whoever holds the link key can read and write the link. Keep it in the platform keystore. See [Storing the link key](guides/key-storage.md). |
| **Pairing** | The link key is only as secret as the way your app delivers it. A QR code shown on one screen and scanned by the other, or an authenticated key exchange, is sound; a key sent through a third channel is only as safe as that channel. See [Pairing devices](guides/pairing.md). |
| **Availability** | A relay can drop envelopes, refuse service or erase a mailbox. Receivers notice gaps in sequence numbers, but cannot force delivery. |
| **A faked mailbox reset** | A relay can pretend a mailbox was reset. If your app then resets its receive position, the relay can deliver old, genuine envelopes a second time. It still cannot forge or change them. See [The mailbox was reset](guides/link-lifecycle.md#the-mailbox-was-reset). |
| **Traffic analysis across links** | The relay cannot tell which mailboxes belong to one user, but requests from the same IP address at the same time hint at it. |

## Cryptography

| Purpose | Algorithm |
|---|---|
| Key derivation | HKDF-SHA256 (RFC 5869), salt = link id, one label per derived value |
| Encryption | ChaCha20-Poly1305 (RFC 8439), 96-bit random nonce per envelope |
| Token digest | SHA-256 |

Each direction has its own key, so the two devices never encrypt under the same key. Random nonces
mean a device that lost its state cannot reuse a nonce, even if it reuses a sequence number. The
details are in [Keys and derivation](protocol/keys.md) and [Envelope format](protocol/envelope.md).

The SDKs use the platform's cryptography (CryptoKit, `javax.crypto`, WebCrypto,
`System.Security.Cryptography`), except ChaCha20-Poly1305 in JavaScript, which comes from the
audited `@noble/ciphers`. Every implementation is tested against vectors from an independent
reference implementation; see [Test vectors](protocol/test-vectors.md).

## Running your own relay

A relay you run is part of your attack surface for availability and metadata, not for content.

- Serve it only over HTTPS. Without TLS, an observer can copy bearer tokens and then fill, prune or
  erase mailboxes (still not read them).
- Do not enable access logs on the relay or its proxy.
- Keep the operating system and Node up to date.

See [Operating a relay](relay/operations.md).

## Reporting a vulnerability

Open an issue in the affected repository on [GitHub](https://github.com/szymonmichalik). If the
details would let someone attack existing users before a fix is out, use "Report a vulnerability"
under that repository's Security tab instead: the report stays private until the fix is published.

I acknowledge reports within three working days and publish a fix and an advisory once users can
update.
Include the component and version, what you did, and what you saw.

In scope: the protocol, the relay server, the four SDKs and this documentation where it gives
unsafe advice. Out of scope: the security of a relay you host yourself beyond this code (its TLS,
operating system and network), and apps built on ZeroByte.
