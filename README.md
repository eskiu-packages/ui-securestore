<p align="center">
  <img src="https://raw.githubusercontent.com/eskiu-lang/eskiu/main/assets/logo.png" alt="Eskiu, a kitten with a ball of yarn" width="140">
</p>

<h2 align="center">securestore</h2>
<p align="center">Small secrets kept by the platform's secure storage, for Eskiu ui apps.</p>
<p align="center"><sub>Part of the <a href="https://github.com/eskiu-lang">Eskiu</a> project</sub></p>

---

```eskiu
import <securestore>;

secure_set("session", token);
string saved = secure_get("session");   // null when there is none; free it
defer free(saved);
secure_remove("session");
```

| Call | Does |
|---|---|
| `secure_get(key)` | the value kept under `key`, a copy the caller frees, or null |
| `secure_set(key, value)` | keeps `value` under `key`, replacing what was there; false when the platform refused |
| `secure_remove(key)` | forgets `key` |
| `secure_is_secure()` | whether values are kept encrypted by the platform (false on the web, in webview builds, and on Linux without a Secret Service) |
| `secure_allow_insecure(allow)` | lets the web and webview builds keep values in `localStorage`, in the clear; off by default |

Where `secure_is_secure()` is false, `secure_set` returns false and `secure_get` returns null:
nothing is kept, so a secret is never written in the clear behind the app's back. Check
`secure_is_secure()` and offer another way (sign in again each session, keep the token in
memory). An app that accepts the clear for a value calls `secure_allow_insecure(true)` first.

| Platform | Keeps values in | `secure_is_secure()` |
|---|---|---|
| iOS | the Keychain, as generic passwords under the app's bundle id, this device only and readable after the first unlock (so a background refresh can read them) | true |
| Android | the app's private preferences, each value encrypted with AES-GCM under a 256-bit key the Android Keystore makes and keeps; the key never leaves the Keystore, so a backup restored to another device cannot open them (`secure_get` is then null) | true |
| macOS | the Keychain, as generic passwords under the app's bundle id, never synchronized; see below | true |
| Linux | the Secret Service (GNOME Keyring, KWallet) through libsecret, an item per key with the app's id; see below | true with a Secret Service, else false |
| Windows | the registry under `HKEY_CURRENT_USER\Software\Eskiu\SecureStore\<program>`, each value encrypted with DPAPI for the current Windows user | true |
| Web, and the desktop's `--webview` builds | nothing: `secure_set` refuses; after `secure_allow_insecure(true)`, `localStorage`, in the clear | false |

On macOS an app signed with a keychain access group (an Apple team's signature) uses the data
protection keychain, this device only, as on iOS. The ui tool signs ad hoc, and such an app
uses the user's login keychain, which macOS keeps encrypted and opens only to the app that
made the item: after a rebuild changes the app's signature, macOS may ask the user once to let
it read its items again.

On Linux libsecret is loaded at run time (`libsecret-1.so.0`), so the app builds and starts
without it. When the library or a Secret Service is missing (a minimal system, a session
without a keyring), `secure_set` returns false, `secure_get` returns null and
`secure_is_secure()` is false: nothing is ever written in the clear. A locked keyring may ask
the user to unlock it.

On Windows the values are bound to the Windows account: another user, or the same files on
another machine, cannot decrypt them. The registry key is named after the program's file name.

On the web there is no secure store to use. The API is synchronous while WebCrypto is not, and
even a non-extractable WebCrypto key kept in IndexedDB decrypts for any script that runs in the
page, which is the same script that can read `localStorage`. So the web keeps nothing unless the
app opts in with `secure_allow_insecure(true)`; then values go to `localStorage` (under
`eskiu-secure:<key>`) as they are, readable by any script in the page and from the browser's
profile. `secure_remove` works either way, so an app can clear what an older version kept.

### Migrating from 0.1.4

Up to 0.1.4 the web and the webview builds kept values in `localStorage` without asking. Now
`secure_set` returns false and `secure_get` returns null there until the app calls
`secure_allow_insecure(true)`. An app that relied on it either calls that once at start (and
accepts the clear), or treats the web as a platform without a store: values an older version
kept stay in `localStorage` until `secure_remove` takes them away.

The package is in Preview, as ui's device packages are (ui's `docs/stability.md`): `eskiuc
test` runs it against a stand-in for the platform, and the desktops' native side is checked
against the real store by `tools/native_smoke.c`.

It is for tokens and keys, not for data: a value is read whole on each call.

`examples/vault` keeps a session token across launches (`ui run ios`, `ui run android`, `ui run macos`, `ui run linux` or `ui run windows`).


## Documentation

Every public function and type, with its documentation, is in the [API reference](docs/api.md).

## Install

In an app made with `ui new`:

```bash
eskiuc add securestore --git https://github.com/eskiu-packages/ui-securestore --rev v0.2.0
```
## Tests

`eskiuc test` checks the calls against an in-memory stand-in for the platform. `tools/native_smoke.c` checks one desktop's native side against the real store (CI runs it on Linux with GNOME Keyring and on Windows; on a Mac: `clang -fobjc-arc native/macos/securestore.m tools/native_smoke.c -framework Foundation -framework Security`).

## License

MIT
