# Cross-platform — React Native & Flutter

Everything in the bundle and assets ships to the device. Native shell: [android.md](android.md), [ios.md](ios.md).

## React Native (JS/TS)

- **Secrets in the bundle**: `API_KEY/SECRET/...` constants in code, `react-native-config` (`.env` is bundled into the app), keys in `app.json` → extractable from the bundle. There is no "hidden environment variable" on the client.
- **Storage**: `AsyncStorage` / `@react-native-async-storage` holding tokens/PII in plaintext → use `react-native-keychain` / `expo-secure-store`.
- **WebView** (`react-native-webview`): `javaScriptEnabled` + `injectedJavaScript`/`injectedJavaScriptBeforeContentLoaded` with untrusted content; `onMessage` trusting messages; `originWhitelist={['*']}`; `source={{uri}}` from a deeplink/untrusted variable; `allowFileAccess`/`allowingReadAccessToURL`.
- **Deeplink**: `Linking.getInitialURL()` / `Linking.addEventListener('url', ...)` using the URL/params without validation (navigation, auth token in the URL).
- **TLS**: `fetch`/`axios` without pinning (use `react-native-ssl-pinning` or a native pinning config); plaintext HTTP fetch.
- **Native bridge**: a native module exposing sensitive methods to JS; `__DEV__`/devtools still enabled in release.
```bash
grep -rnE "AsyncStorage|react-native-config|injectedJavaScript|onMessage|originWhitelist|Linking\.(getInitialURL|addEventListener)|javaScriptEnabled|allowFileAccess|(api[_-]?key|secret|token|password)\s*[:=]" --include=*.js --include=*.jsx --include=*.ts --include=*.tsx .
```
Known-safe: `react-native-keychain`/`expo-secure-store`; SSL pinning present; `originWhitelist` limited to domains; deeplink params validated.

## Flutter (Dart)

- **Secrets in code/assets**: key constants in `.dart`, files in `assets/` (pubspec assets) → extractable; do **not** use `String fromEnvironment` as if it were secret (it still lives in the binary).
- **Storage**: `shared_preferences` holding tokens/PII (plaintext) → use `flutter_secure_storage`.
- **TLS — common trap**: `HttpClient().badCertificateCallback = (cert, host, port) => true` or Dio `IOHttpClientAdapter(createHttpClient: ...)` (`onHttpClientCreate` in Dio 4) returning a client with `badCertificateCallback = (...) => true` → **accepts any cert (MITM)**. Missing cert pinning (`dio_certificate_pinning`/`http_certificate_pinning`).
- **WebView** (`webview_flutter`): `setJavaScriptMode(JavaScriptMode.unrestricted)` (`JavascriptMode.unrestricted` before v4) + `addJavaScriptChannel`/`JavascriptChannel` trusting content; `loadRequest` from an untrusted URL.
- **Deeplink**: `uni_links` / `go_router` / `app_links` handling URIs without validating host/params.
- **Platform channel**: `MethodChannel` passing untrusted data down to native.
```bash
grep -rnE "badCertificateCallback|shared_preferences|flutter_secure_storage|JavaScriptMode\.unrestricted|JavascriptMode\.unrestricted|JavascriptChannel|addJavaScriptChannel|MethodChannel|uni_links|fromEnvironment|loadRequest|(api[_-]?key|secret|token|password)\s*[:=]" --include=*.dart .
```
Known-safe: `flutter_secure_storage`; cert pinning present; `badCertificateCallback` evaluates correctly (does not return true); JS channel accepts only validated data.

## Cross-platform SCA
- RN: `package.json` + `package-lock.json`; the registry in `.npmrc` points to an internal registry/proxy if the organization has one.
- Flutter: `pubspec.yaml` + `pubspec.lock`; `PUB_HOSTED_URL` points to an internal mirror if there is one; scan plugins for CVEs.
- Native shell: Gradle `maven { url }`, CocoaPods `source` pointing to a controlled mirror.
Scan: `trivy fs --scanners vuln --severity HIGH,CRITICAL .`
