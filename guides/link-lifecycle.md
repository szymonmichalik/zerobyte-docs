# Link lifecycle

A link lives from pairing to unpairing. In between, apps restart, devices go offline for weeks,
apps get reinstalled and relays get moved. This page covers each case.

```
pair ──> claim ──> send / receive / watch ──> erase ──> delete local state
                         │
                         ├── 30 days without a write: mailbox expires (link survives)
                         ├── reinstall on one device: re-pair
                         └── relay moved or reset: the mailbox was reset
```

## Start

After pairing, each device stores the link key, link id and role, and creates a `Link` with
`lastReceived` set to none. The first request claims the mailbox. Both devices may claim; the
second gets `204` and continues.

## Every launch

1. Load the link key, link id, role and `lastReceived`.
2. Derive the credentials and create the `Link`.
3. Start the watch, and pull once when it opens.

There is nothing to restore about sending: the SDK asks the relay for the last stored sequence
before its first send in each process.

## Offline for a while

Nothing to do. Envelopes wait on the relay until the receiver collects them, up to 64 per
direction and 32 MiB per mailbox. While the mailbox is full, the sender gets `quota_exceeded` and
should hold back or coalesce; see [Designing messages](message-design.md).

## Thirty days without a write

The relay erases a mailbox 30 days after its last stored envelope or claim. If one device keeps
writing, the mailbox never expires. If nobody writes for 30 days, it does, and the next request
claims a fresh, empty mailbox under the same id. That is a reset; see below.

If your app may sit idle for longer, send a small message from time to time, for example a weekly
heartbeat from the primary, so the mailbox never expires.

## The mailbox was reset

A mailbox starts again from nothing when it expires, when someone erases it, or when the relay
loses its database or is replaced by a new one at the same address. Afterwards:

- Envelopes that were waiting are gone.
- The relay reports no last sequence, so the sender's next message is sequence 1.
- The receiver still holds the old `lastReceived`, say 57, and refuses sequences 1 to 57 as
  replays. Nothing arrives until the sender passes 57.

### Detect it

At launch, and whenever a pull returns nothing for a while although you expect messages, compare
the relay's last sequence for the direction you read with your `lastReceived`:

```swift
let last = try await relay.lastSequence(direction: link.sends.reversed)
if let lastReceived = await link.lastReceived, (last ?? 0) < lastReceived {
    // the mailbox was reset
}
```

The same call exists in every SDK (`lastSequence`, `LastSequenceAsync`). A relay never reports a
lower last sequence for a mailbox that was not reset: pruning does not lower it.

### Recover

Pick one:

1. **Reset the receive position.** Create the `Link` again with `lastReceived` set to none and
   store that. The next pull delivers everything from sequence 1. Then ask the other device for a
   full snapshot, because whatever was waiting before the reset is lost.
2. **Re-pair.** Generate a new link key and id. This is the cleanest option if your app already
   makes pairing easy.

Option 1 has a cost: a dishonest relay could fake a reset and then replay envelopes the receiver
already accepted, since their sequences are valid again. It cannot forge or alter them, only show
old messages a second time. If replaying old messages could cause harm in your app, give each
message an application-level id or timestamp and ignore ones already applied, or re-pair instead.

### Avoid it

- Keep a link active (a write at least every 30 days).
- When moving a relay to a new host, move its database with it. The Node relay's database is a
  single SQLite file; copy it while the relay is stopped.

## Reinstall on one device

A reinstalled app lost its keystore entries on most platforms (iOS keeps Keychain items across
reinstalls unless you delete them on first launch; do so if you rely on a fresh start). Without
the link key the device cannot join the old link: pair again. The other device should offer
"forget this device", which erases the old mailbox.

## Unpair

On the device where the user unpairs:

1. `link.erase()`. The relay deletes the mailbox and closes every watch on it with `1000 erased`.
2. Delete the link key, link id, role and `lastReceived`.

On the other device, the watch closes with reason `erased`. Treat that as "the other device
unpaired": stop, delete local state, and tell the user. If it was not watching, its next request
gets `401` (the mailbox is no longer claimed), and a claim would create a new empty mailbox. To
tell an unpaired link from a reset one, have your app send an explicit "unpairing" message before
erasing, when the device is online.

If the device is offline when the user unpairs, delete local state anyway. The mailbox expires
30 days after its last write.

## Several companions

Each pair of devices is its own link with its own key, id and mailbox. A primary with three
companions holds three `Link`s and sends each message three times, once per link. There is no
broadcast: the relay cannot tell that three mailboxes belong to one user, by design.

## Moving to another relay

All devices must use the same relay and app id. To move:

1. Ship an app version that reads the relay address from a message, or from your own
   configuration.
2. From the primary, send a message naming the new relay.
3. Both devices switch, keeping the link key. The new relay has no mailbox for them, so the first
   request claims one; sequences start from 1. Reset `lastReceived` on both sides as above.
