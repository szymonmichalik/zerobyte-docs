# JavaScript SDK

[zerobyte-js-sdk](https://github.com/szymonmichalik/zerobyte-js-sdk). Apache-2.0. Package name
`@zerobyte-cloud/sdk`.

| | |
|---|---|
| Runtimes | Browsers with WebCrypto, Node 20+, Deno, Bun, Cloudflare Workers |
| Module | ES module, TypeScript types included |
| Uses | WebCrypto (HKDF, SHA-256), `fetch`, `WebSocket` |
| Dependencies | `@noble/ciphers` (MIT, audited) for ChaCha20-Poly1305, which WebCrypto lacks |

## Install

The package is not on npm yet. Install it from GitHub at a tag:

```sh
npm install github:szymonmichalik/zerobyte-js-sdk#v1.0.0
```

It installs as `@zerobyte-cloud/sdk`:

```js
import { Credentials, Direction, Link, Relay } from "@zerobyte-cloud/sdk";
```

Bundlers (Vite, esbuild, webpack, Rollup) handle it like any ES module package.

## Set up a link

```js
const RELAY_BASE = "https://relay.example.com/v1/";
const APP_ID = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d";

const credentials = await Credentials.derive(linkKey, linkID);   // Uint8Array(32), UUID string
const relay = new Relay({ appID: APP_ID, credentials, base: RELAY_BASE });
const link = new Link({
  credentials,
  sends: Direction.C2P,                 // Direction.P2C on the other device
  relay,
  lastReceived: stored.lastReceived,    // bigint, number or null
});
```

`Credentials.derive` is asynchronous because WebCrypto is. It rejects with
`ZeroByteError` code `invalid_key` if the key is not 32 bytes or the id not 16. The link id may be a
UUID string or a 16-byte `Uint8Array`.

## Sequence numbers are `BigInt`

Sequences are unsigned 64-bit, beyond the safe range of `number`. The SDK returns them as `BigInt`
and accepts `BigInt` or `number`. To store one, convert it to a string and back:

```js
localStorage.setItem("lastReceived", link.lastReceived?.toString() ?? "");
const saved = localStorage.getItem("lastReceived");
const lastReceived = saved ? BigInt(saved) : null;
```

`JSON.stringify` throws on a `BigInt`; convert first.

## Send

```js
const sequence = await link.send(new TextEncoder().encode("hello"));   // bigint
```

Rejects with `ZeroByteError` code `too_large` above 1 MiB, and with `RelayError` for relay failures.

## Receive

```js
async function pull() {
  for (const { sequence, plaintext } of await link.receive()) handle(plaintext);
  save(link.lastReceived);
}
```

## Watch

```js
function watch() {
  let delay = 1000;
  const connect = () => {
    const socket = relay.watch(Direction.P2C, () => pull());   // the direction you read
    socket.addEventListener("open", () => {
      delay = 1000;
      pull();
    });
    socket.addEventListener("close", (event) => {
      if (event.reason === "erased") return onUnpairedRemotely();
      setTimeout(connect, delay + Math.random() * 500);
      delay = Math.min(delay * 2, 60_000);
    });
  };
  connect();
}
```

`relay.watch(direction, onSequence, WebSocketImpl)` opens a WebSocket and returns it. It offers the
subprotocols `zerobyte.v1` and `zerobyte.bearer.<token>`, because browsers cannot set headers on a
WebSocket. `onSequence` receives each announced sequence as a `BigInt`.

`WebSocketImpl` defaults to `globalThis.WebSocket`, present in browsers, Deno, Bun, Workers and
Node 22+. On Node 20 pass one, for example from the `ws` package:

```js
import WebSocket from "ws";
relay.watch(Direction.P2C, pull, WebSocket);
```

In a browser, close the socket on `visibilitychange` to hidden if you want to save resources, and
reconnect when visible.

## Unpair

```js
await link.erase();
await deleteStoredLink();
```

## In a browser

- **CORS.** The relay allows every origin and exposes `X-Relay-Last` and `X-Relay-More`. No proxy
  is needed.
- **The token.** It travels in the `Authorization` header and, on the WebSocket, as a subprotocol.
  The relay never echoes the subprotocol back. Requests use `credentials: "omit"`,
  `cache: "no-store"` and `referrerPolicy: "no-referrer"`.
- **The link key.** Keep it out of `localStorage`, which any script on the origin can read. Store
  it wrapped by a non-extractable WebCrypto key kept in IndexedDB:

  ```js
  // once: a non-extractable wrapping key, stored in IndexedDB (structured clone keeps it opaque)
  const wrapKey = await crypto.subtle.generateKey(
    { name: "AES-GCM", length: 256 }, false, ["encrypt", "decrypt"]);

  async function wrap(linkKey) {
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const data = new Uint8Array(await crypto.subtle.encrypt({ name: "AES-GCM", iv }, wrapKey, linkKey));
    return { iv, data };                                   // store this in IndexedDB too
  }

  async function unwrap({ iv, data }) {
    return new Uint8Array(await crypto.subtle.decrypt({ name: "AES-GCM", iv }, wrapKey, data));
  }
  ```

  This keeps the raw key out of storage that scripts can dump, but a script running in your origin
  can still call `unwrap`. A strict Content Security Policy matters more than the wrapping. See
  [Storing the link key](../guides/key-storage.md).

## Custom `fetch`

`Relay` takes a `fetch` option, for example to add a timeout or to route through a test double:

```js
const relay = new Relay({
  appID: APP_ID,
  credentials,
  base: RELAY_BASE,
  fetch: (url, init) => fetch(url, { ...init, signal: AbortSignal.timeout(30_000) }),
});
```

## API reference

```ts
export const Direction: { readonly P2C: 1; readonly C2P: 2 };

export class Credentials {
  static derive(linkKey: Uint8Array, linkID: Uint8Array | string): Promise<Credentials>;
  readonly mailboxID: Uint8Array;       // 16 bytes
  readonly accessToken: Uint8Array;     // 32 bytes
  readonly mailboxName: string;         // 32 lowercase hex characters
  readonly bearerToken: string;         // 43 characters, base64url
  readonly tokenDigest: string;         // 64 lowercase hex characters
}

export class Link {
  constructor(options: {
    credentials: Credentials;
    sends: 1 | 2;
    relay: RelayTransport;
    lastReceived?: bigint | number | null;
  });
  readonly sends: 1 | 2;
  readonly receives: 1 | 2;
  lastReceived: bigint | null;
  send(plaintext: Uint8Array): Promise<bigint>;
  receive(): Promise<{ sequence: bigint; plaintext: Uint8Array }[]>;
  erase(): Promise<void>;
}

export class Relay implements RelayTransport {
  constructor(options: { appID: string; credentials: Credentials; base: string; fetch?: typeof fetch });
  readonly mailbox: string;
  watch(direction: 1 | 2, onSequence: (sequence: bigint) => void,
        WebSocketImpl?: typeof WebSocket): WebSocket;
  // RelayTransport methods
}

export interface RelayTransport {
  claim(): Promise<void>;
  put(envelope: Uint8Array, direction: 1 | 2, sequence: bigint): Promise<void>;
  fetch(direction: 1 | 2, after: bigint | null): Promise<{ envelopes: Uint8Array[]; more: boolean }>;
  lastSequence(direction: 1 | 2): Promise<bigint | null>;
  prune(direction: 1 | 2, through: bigint): Promise<void>;
  erase(): Promise<void>;
}

export function seal(plaintext: Uint8Array,
  options: { direction: 1 | 2; sequence: bigint | number; credentials: Credentials }): Uint8Array;
export function open(envelope: Uint8Array,
  options: { direction: 1 | 2; after?: bigint | number | null; credentials: Credentials }):
  { sequence: bigint; plaintext: Uint8Array };
export function readHeader(envelope: Uint8Array): { direction: 1 | 2; sequence: bigint };
export function joinWire(envelopes: Uint8Array[]): Uint8Array;
export function splitWire(body: Uint8Array): Uint8Array[];

export function hex(bytes: Uint8Array): string;
export function fromHex(text: string): Uint8Array;
export function uuidBytes(uuid: string): Uint8Array;

export class ZeroByteError extends Error { readonly code: string }
export class RelayError extends Error {
  readonly code: string; readonly status: number; readonly last: bigint | null;
}

export const VERSION: 1, HEADER_LENGTH: 10, NONCE_LENGTH: 12, TAG_LENGTH: 16, OVERHEAD: 38;
export const MAX_PLAINTEXT_BYTES: number, MAX_ENVELOPE_BYTES: number, FETCH_LIMIT: 16;
```

`Link` runs its operations one at a time on an internal promise chain, so calling `receive` from
several event handlers at once is safe.

## Testing

```sh
npm install
npm test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> npm test
```
