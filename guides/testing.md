# Testing your integration

Three levels, from fastest to most realistic:

1. **Unit tests with an in-memory relay.** Two `Link`s talk through an object in your test. No
   network, no server, runs in milliseconds.
2. **Integration tests against a local relay.** The real Node relay on `127.0.0.1`.
3. **Manual tests on devices** against a relay you host.

## An in-memory relay

Every SDK's `Link` takes a transport interface (`RelayTransport`, `IRelayTransport`), not a
concrete `Relay`. An implementation that keeps envelopes in a dictionary and follows the relay's
rules is enough for most tests.

The rules to reproduce:

- `put` refuses a sequence not above the last one ever stored in that direction, with a
  sequence conflict carrying that last sequence.
- `fetch` returns envelopes above `after`, oldest first, at most 16, and whether more are waiting.
- `lastSequence` returns the highest sequence ever stored, not the highest still waiting.
- `prune` removes envelopes up to and including `through`.
- `erase` removes everything, including the last sequence.

Swift:

```swift
actor MemoryRelay: RelayTransport {
    private var boxes: [Direction: [(UInt64, Data)]] = [:]
    private var last: [Direction: UInt64] = [:]

    func claim() async throws {}

    func put(_ envelope: Data, direction: Direction, sequence: UInt64) async throws {
        if let previous = last[direction], sequence <= previous {
            throw RelayError.sequenceConflict(last: previous)
        }
        boxes[direction, default: []].append((sequence, envelope))
        last[direction] = sequence
    }

    func fetch(direction: Direction, after sequence: UInt64?) async throws
        -> (envelopes: [Data], more: Bool)
    {
        let waiting = (boxes[direction] ?? []).filter { $0.0 > (sequence ?? 0) }
        return (waiting.prefix(16).map(\.1), waiting.count > 16)
    }

    func lastSequence(direction: Direction) async throws -> UInt64? { last[direction] }

    func prune(direction: Direction, through sequence: UInt64) async throws {
        boxes[direction]?.removeAll { $0.0 <= sequence }
    }

    func erase() async throws {
        boxes = [:]
        last = [:]
    }
}
```

JavaScript:

```js
import { RelayError } from "@zerobyte-cloud/sdk";

class MemoryRelay {
  boxes = new Map();   // direction -> [[sequence, envelope]]
  last = new Map();    // direction -> bigint

  async claim() {}

  async put(envelope, direction, sequence) {
    const previous = this.last.get(direction);
    if (previous !== undefined && BigInt(sequence) <= previous) {
      throw new RelayError("sequence_conflict", 409, previous);
    }
    const box = this.boxes.get(direction) ?? [];
    box.push([BigInt(sequence), envelope]);
    this.boxes.set(direction, box);
    this.last.set(direction, BigInt(sequence));
  }

  async fetch(direction, after) {
    const waiting = (this.boxes.get(direction) ?? []).filter(([s]) => s > BigInt(after ?? 0));
    return { envelopes: waiting.slice(0, 16).map(([, e]) => e), more: waiting.length > 16 };
  }

  async lastSequence(direction) { return this.last.get(direction) ?? null; }

  async prune(direction, through) {
    this.boxes.set(direction, (this.boxes.get(direction) ?? []).filter(([s]) => s > BigInt(through)));
  }

  async erase() { this.boxes.clear(); this.last.clear(); }
}
```

`Link` reads the error's `code` and `last` to continue after a conflict.

A test then pairs two links on one relay:

```swift
@Test func roundTrip() async throws {
    let key = Data((0..<32).map { _ in UInt8.random(in: 0...255) })
    let credentials = try Credentials(linkKey: key, linkID: UUID())
    let relay = MemoryRelay()
    let primary = Link(credentials: credentials, sends: .p2c, transport: relay)
    let companion = Link(credentials: credentials, sends: .c2p, transport: relay)

    try await primary.send(Data("one".utf8))
    try await primary.send(Data("two".utf8))

    let received = try await companion.receive()
    #expect(received.map(\.plaintext) == [Data("one".utf8), Data("two".utf8)])
    #expect(try await companion.receive().isEmpty)
}
```

## Cases worth testing

| Case | How |
|---|---|
| Messages arrive in order, once | Send several, receive, receive again: the second is empty |
| More than one page | Send 40, receive: all 40 arrive in one `receive` call |
| Restart | Create a new receiving `Link` with the stored `lastReceived`: nothing is delivered twice |
| A second writer | Two sending `Link`s on the same side: both succeed, sequences do not collide |
| A forged envelope | Put random bytes under the next sequence: `receive` skips it, prunes it, and delivers what follows |
| A replay | Put an old envelope again under a higher sequence: `receive` refuses it |
| A full mailbox | Make your memory relay throw `quota_exceeded` after N envelopes: your app holds back |
| A reset | `erase` the memory relay and send from 1: your reset detection fires |
| Unpair | `erase` from one side: the other side's next request and your watch handling behave |

Forged and replayed envelopes need a way to write raw envelopes into the memory relay that
bypasses the sequence check, as a hostile relay would. The SDKs' own test suites do exactly this;
their `LinkTests` are a good template.

## Against a local relay

Start the Node relay with a test app id:

```sh
git clone --branch v1.0.0 https://github.com/szymonmichalik/zerobyte-relay.git
cd zerobyte-relay
npm install
APP=$(npm run --silent app-id)
RELAY_APPS=$APP RELAY_DATA=:memory: PORT=8080 npm start
```

`RELAY_DATA=:memory:` keeps the database in memory, so each start is clean.

Point your tests at it with two environment variables, the same ones the SDKs' own suites read:

| Variable | Value |
|---|---|
| `RELAY_BASE` | `http://127.0.0.1:8080/v1/` |
| `RELAY_APP` | The app id the relay was started with |

```sh
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=$APP swift test
```

Skip these tests when the variables are absent, so the suite still runs on a machine without a
relay. Plain HTTP is acceptable only on the loopback address. On iOS, App Transport Security allows
`http://127.0.0.1` from the simulator; on Android, the emulator reaches the host at `10.0.2.2` and
needs a network security config that allows cleartext to that address in debug builds only.

## The SDKs' own tests

To check an SDK on your platform, run its suite. Each one loads the shared test vectors and, with
`RELAY_BASE` and `RELAY_APP` set, also runs against a live relay:

| SDK | Command |
|---|---|
| Swift | `swift test` |
| Android | `./gradlew test` |
| JavaScript | `npm test` |
| .NET | `dotnet test` |
| Relay | `npm test` |

## Devices

On real devices, test what an in-memory relay cannot show:

- the app in the background and suspended, then brought back (the watch reconnects and pulls);
- airplane mode during a send (the send fails cleanly; a later send succeeds);
- one device off for a few days while the other keeps writing (it catches up in one pull, or your
  app handles `quota_exceeded`);
- unpairing from each side.
