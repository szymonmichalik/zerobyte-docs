# Contributing

Corrections and additions are welcome, in this documentation and in the code repositories.

## Where to contribute

| Change | Repository |
|---|---|
| A mistake or a gap in these pages | [zerobyte-docs](https://github.com/szymonmichalik/zerobyte-docs) |
| The protocol or the test vectors | [zerobyte-protocol](https://github.com/szymonmichalik/zerobyte-protocol) |
| The relay | [zerobyte-relay](https://github.com/szymonmichalik/zerobyte-relay) |
| An SDK | [Swift](https://github.com/szymonmichalik/zerobyte-swift-sdk), [Android](https://github.com/szymonmichalik/zerobyte-android-sdk), [JavaScript](https://github.com/szymonmichalik/zerobyte-js-sdk), [.NET](https://github.com/szymonmichalik/zerobyte-dotnet-sdk) |

Security issues go through the process in [Security](SECURITY.md#reporting-a-vulnerability).

## How

1. For anything beyond a small fix, open an issue first and describe the change.
2. Fork the repository and create a branch from `master`.
3. Make the change, with tests for code.
4. Open a pull request against `master`. Changes reach `master` only through pull requests.

## Documentation style

- English, plain and direct. Short sentences. Second person ("you") for instructions.
- No emoji, no marketing language.
- One topic per page. Link to the page that owns a topic instead of repeating it.
- Every code sample must run as written against the current release. Use
  `https://relay.example.com/v1/` as the relay and a placeholder app id.
- Use tables for reference material (status codes, settings, limits).
- Lines wrap at about 100 characters.
- Relative links between pages, with the `.md` extension, so they work on GitHub and in any
  Markdown viewer.

## Protocol changes

The protocol is versioned. Version 1 is frozen: changes to it are limited to clarifications that
do not change what any implementation does. Anything else is a proposal for a future version and
needs an issue in zerobyte-protocol first, with:

- the problem it solves;
- the exact wire and API change;
- how version 1 clients and relays keep working.

Test vectors are produced only by `spec/vectors.py` in zerobyte-protocol and copied into every
implementation. Never regenerate them from an implementation under test.

## Code

Each repository's README lists its build and test commands. Pull requests must pass the test
suite, including the shared vectors. Do not add dependencies without discussing it in an issue
first: the SDKs deliberately depend on the platform alone, with one audited exception in
JavaScript.

## Licence of contributions

By contributing you agree that your contribution is licensed under the licence of the repository
you contribute to: CC-BY-4.0 for this documentation (code samples also Apache-2.0), CC-BY-4.0 for
the protocol text, AGPL-3.0-only for the relay, Apache-2.0 for the SDKs.
