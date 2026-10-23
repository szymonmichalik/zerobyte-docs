# Client behaviour

The relay enforces very little. Correct delivery depends on clients following these rules. The
SDKs' `Link` class implements all of them; read this page if you write your own client or want to
know what `Link` does.

## Claiming

- Claim the mailbox before the first other request, once per process. Claiming is idempotent:
  `201` and `204` are both success.
- Both devices may claim. Whichever claims first creates the mailbox; the other gets `204`.
- A `403` means another token owns the mailbox. With correctly derived credentials this happens
  only if two different links share a mailbox id, which requires the same link key and link id.
  Treat it as a pairing error.
- After an erase, the mailbox is free again. The next request that needs it claims afresh.

## Sending

1. Pick the next sequence:
   - if you sent before in this process, the last sequence you stored plus 1;
   - otherwise, `HEAD` the direction and use `X-Relay-Last` plus 1, or 1 if the header is absent.
2. Seal the plaintext with a fresh random nonce and `PUT` it under that sequence.
3. On `409`, another instance of this side wrote meanwhile (a second install, or a process that
   restarted). Read `X-Relay-Last`, continue from it plus 1, **seal again with a new nonce** and
   retry. Give up after a few attempts.
4. On `507`, the other device has not collected for a while. Do not retry in a loop. Wait until it
   catches up, or drop what you can reconstruct later. See
   [Designing messages](../guides/message-design.md).

Do not persist the next sequence. `HEAD` gives it to you at the start of each process, and a
persisted value can be wrong after a restore from backup.

## Receiving

```
cursor = last_received                    # persisted; null on a fresh link
seen   = null
loop:
    page = GET <dir>?after=cursor or 0&limit=16
    for envelope in page:
        if header cannot be read: skip
        cursor = max(cursor, header.sequence)
        seen   = max(seen,   header.sequence)
        try open(envelope, after=last_received):
            deliver plaintext
            last_received = header.sequence
        except: skip                      # forged, replayed or damaged
    if X-Relay-More is 0 or cursor did not move: break
if seen is not null:
    DELETE <dir>?through=seen
persist last_received
```

Points that are easy to get wrong:

- **Prune everything you saw, including envelopes that failed to open.** An envelope that fails
  authentication now will fail forever. Leaving it on the relay blocks one of the 64 slots until the
  mailbox expires.
- **Advance the fetch cursor past failed envelopes**, or the next page returns the same envelope
  again and the loop never ends. Stop if a page does not move the cursor.
- **Pass the last accepted sequence to `open`**, not the cursor. The replay check is against what
  you accepted.
- **Persist `last_received` after delivering, not before.** If the app crashes between the two,
  you get the messages again on the next run instead of losing them. Make delivery idempotent or
  store `last_received` in the same transaction as the delivered data.

## Persistence

| State | Persist | Why |
|---|---|---|
| Link key, link id | Yes, in the platform keystore | Without them the link is gone |
| Role (primary or companion) | Yes | Decides the direction |
| `last_received` | Yes | Without it, old envelopes still on the relay would be accepted again |
| Next sequence to send | No | `HEAD` recovers it |
| Claimed flag | No | Claiming again is cheap and idempotent |

If `last_received` is lost, the worst case is that envelopes not yet pruned are delivered a second
time. Envelopes already pruned are gone and cannot be replayed.

## Watching

Open a watch on the direction you read while the app is in the foreground, fetch when it opens,
fetch on each message, reconnect with backoff. See [Watch WebSocket](watch.md).

## Unpairing

When the user unpairs, erase the mailbox, then delete the link key and `last_received`. Erasing
closes the other device's watch with `1000 erased`, which it should take as a signal to unpair
too. If the device is offline when the user unpairs, delete the local key anyway; the mailbox
expires 30 days after its last write or claim.

## Retrying

| Error | Retry? |
|---|---|
| Network failure, `5xx` other than `507` | Yes, with exponential backoff |
| `409` | Yes, immediately, from `X-Relay-Last` + 1 with a new nonce |
| `507` | Not in a loop; wait for the other side to collect |
| `413` | No; the message is too large. Split it. |
| `401` on a claimed link | Claim again once. If it still fails, the mailbox was erased and claimed by another token, or the credentials are wrong. |
| `403` on claim | No; pairing error |
| `400`, `404`, `405` | No; a client bug or a wrong base URL or app id |
