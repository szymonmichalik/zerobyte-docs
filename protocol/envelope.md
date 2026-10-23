# Envelope format

An envelope is one sealed message.

```
offset  length  field
0       1       version      0x01
1       1       direction    0x01 p2c, 0x02 c2p
2       8       sequence     UInt64 big-endian, 1 ... 2^64 - 1
10      12      nonce        random
22      n       ciphertext   ChaCha20-Poly1305 (RFC 8439)
22 + n  16      tag
```

The first 10 bytes are the **header**. The total length is `n + 38`.

| Constant | Value |
|---|---|
| Header | 10 bytes |
| Nonce | 12 bytes |
| Tag | 16 bytes |
| Overhead | 38 bytes |
| Maximum plaintext | 1,048,576 bytes (1 MiB) |
| Maximum envelope | 1,048,614 bytes |
| Minimum envelope | 38 bytes (an empty plaintext is valid) |

## Sealing

```
key   = p2c_key if direction == 1 else c2p_key
nonce = 12 random bytes from a secure generator
aad   = header ‖ mailbox_id                         # 10 + 16 = 26 bytes
ciphertext ‖ tag = ChaCha20-Poly1305-Encrypt(key, nonce, plaintext, aad)
envelope = header ‖ nonce ‖ ciphertext ‖ tag
```

The **additional authenticated data** is the 10-byte header followed by the 16-byte mailbox id. So
an envelope cannot be moved to another mailbox, flipped to the other direction or renumbered:
any of these changes makes the tag fail.

The nonce is random rather than derived from the sequence. A sender that lost its state, for
example after a reinstall, may reuse a sequence number that the relay then refuses; with a random
nonce it never reuses a nonce under the same key. With 96-bit random nonces and at most a few
million envelopes per link, the chance of a collision is negligible.

Refuse plaintext over 1 MiB with `too_large` before sealing.

## Opening

A receiver opens an envelope only if, **in this order**:

| Check | Error code |
|---|---|
| Length is at most 1 MiB + 38 | `too_large` |
| Length is at least 38 | `malformed` |
| Version byte is `1` | `unsupported_version` |
| Direction byte is `1` or `2` | `malformed` |
| Direction is the one this receiver reads | `wrong_direction` |
| Sequence is greater than the last one accepted | `replayed` |
| The tag verifies under the direction's key with `aad = header ‖ mailbox_id` | `authentication_failed` |

The order matters for test vectors: an envelope that is both replayed and tampered with reports
`replayed`. Every SDK uses these codes.

Checking the sequence before decrypting is safe: the header is authenticated, so an attacker who
changes the sequence also breaks the tag. Checking it first just avoids decrypting what will be
refused anyway.

## Reading the header without opening

`readHeader` (named `header(of:)` in Swift, `ReadHeader` in .NET) returns the direction and
sequence after the first four checks of the length, version and direction byte. A receiver uses it
to advance its fetch cursor past envelopes that fail to open, and the relay uses the same logic to
check that the header matches the URL.

## Example

From the [test vectors](test-vectors.md), credential set 0, direction p2c, sequence 1, nonce
`070707070707070707070707`, plaintext `"Derived relay v1"`:

```
01                                   version
01                                   direction p2c
0000000000000001                     sequence 1
070707070707070707070707             nonce
fa01b53aaf28cacff683529ef...         ciphertext ‖ tag
```

The fixed nonce exists only for known-answer tests. Every SDK's public `seal` draws a random
nonce; the overload that takes a nonce is internal or documented as test-only.
