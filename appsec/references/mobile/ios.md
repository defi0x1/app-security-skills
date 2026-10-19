# iOS — security patterns (Swift/Obj-C)

## Info.plist — ATS & URL scheme
- `NSAppTransportSecurity` → `NSAllowsArbitraryLoads = true` (disables ATS globally) or `NSExceptionAllowsInsecureHTTPLoads = true` for a domain → allows plaintext HTTP.
- `NSAllowsLocalNetworking`, `NSAllowsArbitraryLoadsInWebContent`.
- `CFBundleURLTypes`/`CFBundleURLSchemes` (custom URL scheme) → the app must validate `sourceApplication`/params when opened; an arbitrary scheme is easily invoked by other apps.
```bash
grep -nE 'NSAllowsArbitraryLoads|NSExceptionAllowsInsecureHTTPLoads|NSAllowsLocalNetworking|CFBundleURLSchemes|NSAllowsArbitraryLoadsInWebContent' Info.plist
```
Known-safe: ATS enabled by default (no keys); exceptions with a justification and limited to specific domains.

## TLS / pinning
- `URLSession` delegate `urlSession(_:didReceive:completionHandler:)` calling `completionHandler(.useCredential, URLCredential(trust: challenge.protectionSpace.serverTrust!))` unconditionally → accepts any cert (MITM).
- Missing certificate/public-key pinning on sensitive channels.
```bash
grep -rnE 'didReceive challenge|serverTrust|\.useCredential|URLCredential\(trust:|SecTrustEvaluate' --include=*.swift --include=*.m .
```
Known-safe: pinning present (compares public key/cert), or correct `SecTrustEvaluateWithError` evaluation.

## Keychain / storage
- Storing secrets/tokens in `UserDefaults`/plist/files instead of the Keychain.
- Keychain `kSecAttrAccessibleAlways` (deprecated) flag always; items without `...ThisDeviceOnly` migrate to other devices via backup. `...AfterFirstUnlock` is acceptable when background access is needed; for secrets used only in the foreground prefer `...WhenUnlockedThisDeviceOnly`.
- Core Data / SQLite / plaintext files containing PII.
```bash
grep -rnE 'UserDefaults|NSUserDefaults|kSecAttrAccessibleAlways|kSecAttrAccessibleAfterFirstUnlock|writeToFile|FileManager\.default\.createFile' --include=*.swift --include=*.m .
```
Known-safe: Keychain with `...WhenUnlockedThisDeviceOnly` or `...WhenPasscodeSetThisDeviceOnly`; correct Data Protection class.

## WKWebView
- `WKWebView` with JavaScript enabled (default; `allowsContentJavaScript`) loading untrusted content.
- `evaluateJavaScript(<string containing untrusted data>)`.
- `WKScriptMessageHandler` (`userContentController(_:didReceive:)`) trusting messages from JS → a bridge into native.
- `loadFileURL(_:allowingReadAccessTo:)` with an overly broad directory; `UIWebView` (deprecated, rejected by App Store submission).
```bash
grep -rnE 'WKWebView|UIWebView|evaluateJavaScript|WKScriptMessageHandler|didReceive message|loadFileURL|loadHTMLString' --include=*.swift --include=*.m .
```

## Crypto
- CommonCrypto using `CC_MD5`, `CC_SHA1`, `kCCAlgorithmDES`, ECB (`kCCOptionECBMode`); hardcoded key; static IV; `random()`/`rand()` for keys (use `SecRandomCopyBytes`; `arc4random*` is a CSPRNG on Apple platforms and is not a finding).
```bash
grep -rnE 'CC_MD5|CC_SHA1|kCCAlgorithm(DES|RC4)|kCCOptionECBMode|SecRandomCopyBytes|\brandom\(\)|\brand\(\)' --include=*.swift --include=*.m .
```

## Pasteboard / leakage / logging
- `UIPasteboard.general` holding sensitive data (the clipboard is shared across apps).
- Secret-entry TextField missing `isSecureTextEntry = true` (keyboard cache).
- Screen not covered when entering background (snapshot exposes data).
- `print(...)` / `NSLog(...)` / `os_log` containing secrets/PII.
```bash
grep -rnE 'UIPasteboard|isSecureTextEntry|NSLog\(|os_log\(|print\(' --include=*.swift --include=*.m .
```

## Embedded secrets
```bash
grep -rnE '(apiKey|secret|token|password|AIza[0-9A-Za-z_-]{20,}|sk_live_)' --include=*.swift --include=*.m --include=*.plist --include=*.xcconfig . 2>/dev/null
```
Known-safe: injected at build time from a CI secret (not committed), or fetched at runtime from the server over a pinned channel.
