# Storing the link key

Whoever holds the link key can read every message on the link and write messages the other device
will accept. Store it like a password: in the platform's keystore, on this device only, never in a
synchronised or backed-up location.

## What to store

| Item | Secret | Where |
|---|---|---|
| Link key (32 bytes) | Yes | Platform keystore |
| Link id (16 bytes) | No | Next to the key, or in app storage |
| Role (primary or companion) | No | App storage |
| `lastReceived` | No | App storage, ideally in the same transaction as the data it delivered |

You do not need to store the derived values (mailbox name, token, direction keys): deriving them
takes microseconds.

## Per platform

| Platform | Recommended storage | Notes |
|---|---|---|
| iOS, iPadOS, watchOS, visionOS | Keychain, `kSecClassGenericPassword`, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`, `kSecAttrSynchronizable = false` | Available to background tasks after the first unlock; never leaves the device |
| macOS | Keychain (data protection keychain with `kSecUseDataProtectionKeychain = true`, same attributes) | Same as iOS |
| Android | App-private file, encrypted with an Android Keystore AES-GCM key | Exclude from cloud backup and device transfer in `dataExtractionRules` |
| Windows | DPAPI, `ProtectedData.Protect` with `DataProtectionScope.CurrentUser` | Bound to the Windows user account |
| Linux desktop | Secret Service via libsecret | Fall back to a file readable only by the user if no keyring is running |
| Browser | Wrapped by a non-extractable WebCrypto key in IndexedDB | See below |
| Server | A secrets manager or an encrypted file with restricted permissions | A server holding a link key is a device of the link |

The [Swift](../sdk/swift.md#store-the-link-key), [Android](../sdk/android.md#store-the-link-key),
[JavaScript](../sdk/javascript.md#in-a-browser) and [.NET](../sdk/dotnet.md#store-the-link-key)
pages have code.

## Why "this device only"

Synchronised keychains and cloud backups copy the key to other devices and to the cloud provider.
A restored backup on a new phone would hold a key for a link the user thinks belongs to the old
phone, and both would read the same messages. Re-pairing after moving to a new device is the
correct behaviour.

## Browsers

A browser has no keystore that protects a secret from scripts running in the same origin.
Wrapping the key with a non-extractable `CryptoKey` keeps it out of `localStorage` dumps and out
of IndexedDB exports, but any script in the origin can still ask WebCrypto to unwrap it. The real
protection is keeping hostile scripts out:

- a strict Content Security Policy with no `unsafe-inline` and no third-party scripts;
- Subresource Integrity on anything loaded from elsewhere;
- serving the app from its own origin, not one shared with other content.

For a higher bar, derive the wrapping key from a passphrase the user types on each visit, with
PBKDF2 or Argon2, and keep nothing that can unwrap it at rest.

## In memory

Keep the key in memory only while a `Link` needs it. The SDKs keep the derived direction keys in
the `Credentials` object; dropping the object drops them. Platform garbage collectors do not
guarantee that memory is zeroed, which is acceptable for most apps.

## On unpair

Delete the key, the link id, the role and `lastReceived`, after erasing the mailbox. If the erase
fails because the device is offline, delete the local values anyway: the mailbox expires on its
own after 30 days without writes.
