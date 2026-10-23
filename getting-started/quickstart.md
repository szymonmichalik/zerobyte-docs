# Quickstart

This runs a relay on your machine and has two "devices", two Node scripts, exchange messages
through it. It takes about ten minutes and needs Node 22.13 or newer and git.

Every SDK follows the same steps; once this works, the [SDK pages](../sdk/README.md) show the same
code in Swift, Kotlin and C#.

## 1. Start a relay

```sh
git clone https://github.com/szymonmichalik/zerobyte-relay.git
cd zerobyte-relay
npm run app-id
```

`app-id` prints a fresh UUIDv7, for example `019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d`. Keep it: it
is your app id. Start the relay with it registered:

```sh
RELAY_APPS=<app id> RELAY_DATA=./data/relay.sqlite PORT=8080 npm start
```

It prints one line and then nothing else, because it never logs requests:

```
ZeroByte Relay v1 on 0.0.0.0:8080, 1 app(s), data in ./data/relay.sqlite
```

Leave it running and open a second terminal.

## 2. Create a project

```sh
mkdir zerobyte-quickstart && cd zerobyte-quickstart
npm init -y
npm pkg set type=module
npm install github:szymonmichalik/zerobyte-js-sdk#v1.0.0
```

The package installs as `@zerobyte-cloud/sdk`.

## 3. Pair

Real devices agree the link key during pairing (see [Pairing devices](../guides/pairing.md)). Here
both scripts read it from one file. Create `pair.mjs`:

```js
import { writeFileSync } from "node:fs";

const linkKey = crypto.getRandomValues(new Uint8Array(32));
writeFileSync(
  "link.json",
  JSON.stringify({
    linkKey: Buffer.from(linkKey).toString("base64"),
    linkID: crypto.randomUUID(),
  }),
);
console.log("Paired: link.json written");
```

```sh
node pair.mjs
```

## 4. Shared setup

Create `link.mjs`, which both devices use. Replace `<app id>` with yours.

```js
import { readFileSync, writeFileSync, existsSync } from "node:fs";
import { Credentials, Link, Relay } from "@zerobyte-cloud/sdk";

const APP_ID = "<app id>";
const BASE = "http://127.0.0.1:8080/v1/";

export async function open(sends, stateFile) {
  const pairing = JSON.parse(readFileSync("link.json", "utf8"));
  const credentials = await Credentials.derive(
    new Uint8Array(Buffer.from(pairing.linkKey, "base64")),
    pairing.linkID,
  );
  const relay = new Relay({ appID: APP_ID, credentials, base: BASE });
  const saved = existsSync(stateFile) ? readFileSync(stateFile, "utf8") : null;
  const link = new Link({
    credentials,
    sends,
    relay,
    lastReceived: saved ? BigInt(saved) : null,
  });
  const save = () => {
    if (link.lastReceived !== null) writeFileSync(stateFile, link.lastReceived.toString());
  };
  return { relay, link, save };
}
```

`lastReceived` is the one piece of state a device must keep between runs. Here it goes to a file;
in an app it goes next to the rest of your app's state.

## 5. The companion listens

Create `companion.mjs`:

```js
import { Direction } from "@zerobyte-cloud/sdk";
import { open } from "./link.mjs";

const { relay, link, save } = await open(Direction.C2P, "companion.state");

async function pull() {
  for (const { sequence, plaintext } of await link.receive()) {
    console.log(`#${sequence}: ${new TextDecoder().decode(plaintext)}`);
  }
  save();
}

const socket = relay.watch(Direction.P2C, () => pull());
socket.addEventListener("open", () => pull());
socket.addEventListener("close", () => console.log("watch closed"));
console.log("Companion is watching. Ctrl-C to stop.");
```

```sh
node companion.mjs
```

## 6. The primary sends

In a third terminal, create `primary.mjs`:

```js
import { Direction } from "@zerobyte-cloud/sdk";
import { open } from "./link.mjs";

const { link } = await open(Direction.P2C, "primary.state");
const text = process.argv[2] ?? "hello";
const sequence = await link.send(new TextEncoder().encode(text));
console.log(`sent #${sequence}`);
```

```sh
node primary.mjs "hello from the primary"
node primary.mjs "and a second one"
```

The companion prints each message as it arrives:

```
#1: hello from the primary
#2: and a second one
```

Stop the companion, send two more messages, and start it again: it fetches both when the socket
opens, because the relay held them.

## 7. Unpair

Erasing the mailbox deletes everything in it for both devices. Create `unpair.mjs`:

```js
import { Direction } from "@zerobyte-cloud/sdk";
import { open } from "./link.mjs";

const { link } = await open(Direction.P2C, "primary.state");
await link.erase();
console.log("mailbox erased");
```

The companion's watch socket closes when the mailbox is erased.

## What just happened

- Both scripts derived the same mailbox id, bearer token and keys from `link.json`. The relay saw
  only the mailbox id, the token and the sealed envelopes.
- The first request from either device claimed the mailbox for that token.
- The primary asked the relay for the last sequence it had stored (none), then sent 1, 2, ...
- The companion fetched after its last received sequence, opened each envelope, saved
  `lastReceived`, and pruned what it read.

## Next

- [Concepts](concepts.md) if a term above was unclear.
- [Running a relay](../relay/README.md) to put the relay on a server with HTTPS.
- Your platform's [SDK page](../sdk/README.md).
- [Pairing devices](../guides/pairing.md) and [Storing the link key](../guides/key-storage.md)
  before you ship.
