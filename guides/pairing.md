# Pairing devices

Pairing gives two devices the same link key and link id, and decides which one is the primary.
ZeroByte does not take part: it starts working once both devices hold the values. The security of
everything after depends on how you do this step.

## What pairing must achieve

1. Both devices end up with the same 32-byte link key, and nobody else has it.
2. Both devices end up with the same 16-byte link id.
3. Each device knows its role: one primary, one companion.
4. The user is sure they paired the two devices they meant to, not a device an attacker slipped in.

## Option 1: QR code

The simplest sound method when one device has a screen and the other a camera.

**The device that shows the code** (usually the companion, for example a laptop):

1. Generates a fresh link key from a secure random generator and a random link id.
2. Shows a QR code with both, and its role.
3. Starts watching the direction it reads, so it notices when the other device writes.

**The device that scans** (usually the phone):

1. Reads the link key, link id and role from the code.
2. Stores them and takes the other role.
3. Sends a first message, for example its name and app version, so the other device knows the
   pairing completed.

A payload format that works:

```
zerobyte:1?k=<link key, base64url>&l=<link id, UUID text>&r=c
```

where `r=c` says the device showing the code is the companion. Any format your app can parse is
fine; the protocol does not define one.

Why this is sound: the key travels only through the light between two screens a user is holding.
An attacker would need to see the code. Show it only while pairing, generate a new key every time
the code is shown, and clear it from the screen as soon as the other device has written.

Do not put a link key in a link that can be shared, such as a URL sent by email or chat. Anyone who
sees the message holds the key.

## Option 2: Key agreement with a short code

When neither device can scan the other, the devices can agree a key over the network with an
ephemeral Diffie-Hellman exchange and have the user compare a short code on both screens.

Outline, with X25519:

1. Each device generates an ephemeral X25519 key pair.
2. They exchange public keys over any channel: a temporary rendezvous, local network discovery,
   Bluetooth. The channel need not be secret.
3. Each computes the shared secret and derives:

   ```
   transcript = SHA-256("zerobyte-pairing-v1" ‖ pubkey_primary ‖ pubkey_companion)
   link_key   = HKDF-SHA256(ikm=shared_secret, salt=transcript, info="link key", length=32)
   code       = HKDF-SHA256(ikm=shared_secret, salt=transcript, info="sas",      length=4)
   ```

4. Both screens show `code` as six decimal digits (for example the first 4 bytes as a big-endian
   integer, modulo 1,000,000).
5. The user confirms the digits match on both devices. If they do not, abort: someone is in the
   middle.
6. The link id can be the first 16 bytes of `transcript`, or a UUID one device chooses and sends.

The short code is what defeats a man in the middle: an attacker who replaced the public keys would
end up with different secrets on each side, so the codes differ. The rendezvous channel must not be
able to choose its keys after seeing yours; the standard way is for one side to commit to its
public key (send its hash) before seeing the other's.

This is the same idea as Bluetooth numeric comparison and Signal's safety numbers. If you are not
comfortable implementing it, use the QR code.

## Option 3: An existing secure channel

If both devices already share a secure, authenticated channel, such as a signed-in account on
your own end-to-end encrypted service, or a wired connection, one device can generate the link key
and send it over that channel. The link is then exactly as safe as that channel.

## Choosing roles

The primary is usually the device that holds the source of truth and the companion the one that
mirrors it. It does not matter which device shows the QR code, as long as the payload says who is
which and the devices never both take the same role. If they did, both would write `p2c` and read
`c2p`, and neither would see the other's messages.

## Re-pairing

Pairing again, for example after a device was reset, should produce a **new** link key and link id.
Erase the old mailbox from the device that still has it if you can. Reusing an old key is not
dangerous by itself, but a new key gives a fresh mailbox and avoids confusion about sequence
numbers.

## One device, several companions

A phone paired with a laptop and a watch has two links, each with its own key, link id and
mailbox. Keep a list of links on the primary, and one `Link` object per companion. The SDKs have
no special support for this; each link is independent.
