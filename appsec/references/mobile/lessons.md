# Mobile: common traps and the correct fix

Supplements [android.md](android.md), [ios.md](ios.md), [cross-platform.md](cross-platform.md). Threat model: the device and the binary are in the attacker's hands. Every control that matters is decided on the server. Also never log tokens/PII in any build (crash and analytics SDKs ship logs out). Flow and boundary bugs (deeplinks bound to a session, OAuth handoff to the wrong app, bridges keyed on origin at call time, exported components after manifest merge, leaks on account switch/logout): [../domains/local-ipc-desktop.md](../domains/local-ipc-desktop.md).

| Topic | Wrong / misconception | Correct |
|---|---|---|
| SharedPreferences | `MODE_PRIVATE` is safe | It is only a file permission per UID; the XML is plaintext. Tokens must be encrypted with an Android Keystore key (e.g. Tink or `EncryptedSharedPreferences`, which is deprecated but not broken) |
| Backup | Not declaring `allowBackup` is fine | Default is `true`: cloud backup and device-to-device transfer copy app data (`adb backup` is blocked for non-debuggable apps targeting API 31+, but the other paths remain). Set `allowBackup="false"` or exclude sensitive files via `fullBackupContent` (API <= 30) / `dataExtractionRules` (API 31+) |
| Custom scheme `myapp://` | Used to carry a login token | Another app can register the same scheme and receive the link. Use App Links / Universal Links (domain-verified) + a one-time code; OAuth for native apps uses PKCE |
| Deeplink into WebView | `webView.loadUrl(uri.getQueryParameter("redirect"))` | Allowlist scheme `https` + host; a WebView with a JS bridge loads trusted origins only |
| `exported` | Set `false` on the Activity that receives links from the browser | An Activity with a BROWSABLE intent-filter must be `exported="true"`; treat all Intent data as untrusted and validate it |
| Pinning | Pin the leaf cert; no backup pin | Pin SPKI (survives renewal with the same key) + at least one backup pin |
| Pinning + hostname | Pin an intermediate CA + `hostnameVerifier { _, _ -> true }` | A valid cert for another domain from the same CA still matches the pin, so MITM works. Never disable hostname verification |
| `debug-overrides` user CA | Worry that the release build is affected | Applies only when `debuggable="true"`; keep the release build non-debuggable |
| Frida removes pinning | Try to make the client "impossible to remove" | On a rooted device every client control can be removed; it only raises cost. Authz, limits, pricing stay on the server (do not report pinning bypass or missing root detection as a finding on its own) |
| Biometric | `onAuthenticationSucceeded` sets a flag, then the token is read from prefs | Without a `CryptoObject` it is just an if branch that Frida can hook. Bind a Keystore key with `setUserAuthenticationRequired(true)` and use a `CryptoObject` |
| iOS Keychain | Switching to `WhenUnlockedThisDeviceOnly` is enough | An unlocked device can still read it. Attach a `SecAccessControl` (e.g. `.biometryCurrentSet`) so the Keychain itself demands authentication |
| Token | Long-lived access token, logout only clears local state | Short-lived access token, refresh token in Keystore/Keychain, logout clears local state **and** revokes on the server (RFC 7009) |
| Sensitive screens | — | `FLAG_SECURE` (blocks screenshots, screen recording, Recents thumbnail); on iOS cover the snapshot when entering background |
| R8/ProGuard | Already obfuscated so it is safe | It only renames and shrinks; jadx reconstructs logic, strings, endpoints. Do not embed secrets (including in `.so` files) |

## Super App, MiniApp, JS Bridge

- Each native method checks which MiniApp is calling (per-MiniApp permissions) and validates parameters; a page with XSS lets malicious script call the bridge.
- `Intent.parseUri(params, URI_INTENT_SCHEME)` + `startActivity` can open any Activity. Correct: only `ACTION_VIEW` with an `https` host in the allowlist.
- `evaluateJavascript("$callbackFn('...')")` with `callbackFn` from the MiniApp is code injection in the Super App's WebView. Correct: a callback ID or a name matching an identifier regex; pass data as JSON-encoded.
- MiniApp bundles from a partner CDN: sign the bundle and verify the signature before loading (pinning does not help when the CDN itself is taken over); the bridge issues a token scoped to each MiniApp, never the Super App's main token.
