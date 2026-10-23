# Keys and derivation

## Inputs

Two paired devices share:

- the **link key** `K`: 32 bytes, secret;
- the **link id** `L`: 16 bytes, not secret. When the link id is a UUID, its 16 bytes are taken in
  RFC 4122 order, the order of the hex digits in its text form.

An implementation must reject a link key that is not exactly 32 bytes or a link id that is not
exactly 16 bytes (`invalid_key`).

## Derived values

Every value is HKDF-SHA256 (RFC 5869) with input keying material `K`, salt `L`, and the UTF-8
label below as `info`:

| Value | `info` | Length | Use |
|---|---|---|---|
| Mailbox id | `derived-relay-mailbox-v1` | 16 | In URLs as 32 lowercase hex characters |
| Access token | `derived-relay-token-v1` | 32 | Bearer, base64url without padding (43 characters) |
| p2c key | `derived-relay-p2c-v1` | 32 | Seals primary to companion |
| c2p key | `derived-relay-c2p-v1` | 32 | Seals companion to primary |

In pseudocode:

```
mailbox_id = HKDF-SHA256(ikm=K, salt=L, info="derived-relay-mailbox-v1", length=16)
token      = HKDF-SHA256(ikm=K, salt=L, info="derived-relay-token-v1",   length=32)
p2c_key    = HKDF-SHA256(ikm=K, salt=L, info="derived-relay-p2c-v1",     length=32)
c2p_key    = HKDF-SHA256(ikm=K, salt=L, info="derived-relay-c2p-v1",     length=32)

mailbox_name = lowercase_hex(mailbox_id)               # 32 characters
bearer       = base64url_no_padding(token)             # 43 characters
token_digest = lowercase_hex(SHA-256(token))           # 64 characters
```

Both devices derive identical values. Only the mailbox name and the bearer leave the device; the
relay keeps only `token_digest`.

## Why each choice

- **Salt = link id.** Two links that, by accident or a broken random generator, used the same link
  key still get different mailboxes and keys, as long as their link ids differ.
- **Separate keys per direction.** An envelope the primary sealed cannot be opened as if the
  companion had sealed it. Each side's sequence space is independent, so neither can replay the
  other's envelopes back to it.
- **Token digest on the relay.** A leak of the relay's storage does not let anyone write to or
  read from a mailbox. The digest is of the 32 raw token bytes, not of the base64url text.
- **Labels with a version.** A future version derives entirely different values, so v1 and v2
  mailboxes never collide.

## Example

From the first credential set in the [test vectors](test-vectors.md):

| Field | Value |
|---|---|
| Link key | `000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f` |
| Link id | `6f1c2a3b4d5e4f60817293a4b5c6d7e8` |
| Mailbox name | `41fa13c183c0c2092a085390c172cba1` |
| Token | `d7a2e6dedfcf173e6da839ffc3c70891b914b63f33af155c3203b8fea48d5fbe` |
| Bearer | `16Lm3t_PFz5tqDn_w8cIkbkUtj8zrxVcMgO4_qSNX74` |
| Token digest | `8da1975827ae6db05a1f3be74348c315975316e3e10c22b12abfa2febc7d8b51` |
| p2c key | `87f0a25819d3e9abebcee86981d60190a2d52a5b4abcb5d328c7dc93c2cf1787` |
| c2p key | `c1a45291ad96b7b3b936f0664347a98f1cb4bc45a7ff701b6c433bea863ff06b` |

You can check a derivation with nothing but Python:

```python
import hashlib, hmac, base64

def hkdf(ikm, salt, info, length):
    prk = hmac.new(salt, ikm, hashlib.sha256).digest()
    out, block, counter = b"", b"", 1
    while len(out) < length:
        block = hmac.new(prk, block + info + bytes([counter]), hashlib.sha256).digest()
        out += block
        counter += 1
    return out[:length]

K = bytes(range(32))
L = bytes.fromhex("6f1c2a3b4d5e4f60817293a4b5c6d7e8")
token = hkdf(K, L, b"derived-relay-token-v1", 32)
print(hkdf(K, L, b"derived-relay-mailbox-v1", 16).hex())
print(base64.urlsafe_b64encode(token).rstrip(b"=").decode())
print(hashlib.sha256(token).hexdigest())
```

## UUID byte order

A UUID's text form `6f1c2a3b-4d5e-4f60-8172-93a4b5c6d7e8` maps to the bytes `6f 1c 2a 3b 4d 5e
4f 60 81 72 93 a4 b5 c6 d7 e8`. Some platforms store UUIDs differently in memory:

| Platform | Correct conversion |
|---|---|
| Swift | `withUnsafeBytes(of: uuid.uuid) { Data($0) }`; the SDK does this |
| Kotlin / Java | most significant long, then least significant long, both big-endian; the SDK does this |
| JavaScript | parse the hex digits in order; `uuidBytes()` in the SDK does this |
| .NET | `guid.ToByteArray(bigEndian: true)`; **not** `ToByteArray()`, which swaps the first three groups |

Every SDK accepts a UUID directly and converts it correctly. If you pass raw bytes instead, make
sure both devices produce the same 16 bytes.
