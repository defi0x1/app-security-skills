# Android — security patterns (Kotlin/Java)

## AndroidManifest.xml — platform surface
- `android:exported="true"` on an activity/service/receiver/**provider** without `permission` / `signature` → externally invokable. An exported provider is especially dangerous (data read/write, SQLi, path traversal).
- `android:allowBackup="true"` (default is true!) → adb backup extracts app data. Flag if the app stores sensitive data.
- `android:debuggable="true"` → runtime debugging/injection. CRITICAL if it reaches a release build.
- `android:usesCleartextTraffic="true"` or missing `networkSecurityConfig` → allows plaintext HTTP.
- `<intent-filter>` with `<data android:scheme=...>` (deeplink) → check that the app validates host/path/param before using them; `android:autoVerify="true"` for App Links.
- Custom `<permission android:protectionLevel="normal">` protecting sensitive functionality → should be `signature`.
- `<provider grantUriPermissions="true">` / missing `<path-permission>` → file leak through content URIs.

```bash
grep -nE 'android:exported="true"|allowBackup="true"|android:debuggable="true"|usesCleartextTraffic="true"|android:scheme|protectionLevel="(normal|dangerous)"|grantUriPermissions' AndroidManifest.xml
grep -L 'networkSecurityConfig' AndroidManifest.xml   # missing network security config
```
Known-safe: `exported="false"`, `signature` permission, a `networkSecurityConfig` that blocks cleartext.

## WebView (RCE / data theft surface)
- `setJavaScriptEnabled(true)` + `addJavascriptInterface(obj, "name")` with untrusted content → JS calls native methods (RCE < API 17; data exfiltration on every version).
- `setAllowFileAccess(true)` / `setAllowFileAccessFromFileURLs(true)` / `setAllowUniversalAccessFromFileURLs(true)` → JS reads local files.
- `loadUrl(<variable controlled by user/intent>)`, `loadDataWithBaseURL` with untrusted HTML.
- `setWebContentsDebuggingEnabled(true)` in a release build.
- `onReceivedSslError { handler.proceed() }` → bypasses TLS errors.

```bash
grep -rnE 'addJavascriptInterface|setJavaScriptEnabled\(true\)|setAllowUniversalAccessFromFileURLs\(true\)|setAllowFileAccess(FromFileURLs)?\(true\)|setWebContentsDebuggingEnabled\(true\)|onReceivedSslError|loadDataWithBaseURL' --include=*.kt --include=*.java .
```

## Network / TLS
- Hand-rolled TrustManager with an empty `checkServerTrusted` (trust-all); `HostnameVerifier { _, _ -> true }`; `SSLSocketFactory` ALLOW_ALL; OkHttp without a `CertificatePinner`.
```bash
grep -rnE 'checkServerTrusted|X509TrustManager|ALLOW_ALL_HOSTNAME_VERIFIER|HostnameVerifier|TrustManager\[\]|setHostnameVerifier|CertificatePinner|sslSocketFactory' --include=*.kt --include=*.java .
```
Known-safe: a `CertificatePinner`/`networkSecurityConfig` pin; the system default trust.

## Crypto
- `Cipher.getInstance("AES")` (defaults to ECB!) or `"AES/ECB/..."`, `"DES"`, `"RC4"`; `MessageDigest.getInstance("MD5"|"SHA-1")` for security purposes; static IV/salt; `SecretKeySpec("hardcoded".toByteArray(), ...)`; `Random()`/`Math.random()` generating keys/tokens/IVs (use `SecureRandom`).
```bash
grep -rnE 'Cipher\.getInstance\("(AES|DES|RC4)("|/ECB)|getInstance\("(MD5|SHA-1)"\)|SecretKeySpec\(|IvParameterSpec\(|new Random\(|Math\.random' --include=*.kt --include=*.java .
```

## Storage & logging
- `getSharedPreferences(..., MODE_WORLD_READABLE|MODE_WORLD_WRITEABLE)`; writing secrets/PII in plaintext to prefs/SQLite/external storage; `Environment.getExternalStorage*` for sensitive data.
- `SQLiteDatabase.rawQuery/execSQL` concatenating user strings → SQLi (especially in a content provider).
- `Log.d/v/i/e(...)` containing tokens/passwords/PII; `printStackTrace` leaks information.
```bash
grep -rnE 'MODE_WORLD_(READABLE|WRITEABLE)|rawQuery\(|execSQL\(|getExternalStorage|Log\.[dveiw]\(|printStackTrace' --include=*.kt --include=*.java .
```
Known-safe: `EncryptedSharedPreferences`, Android Keystore (`KeyStore.getInstance("AndroidKeyStore")`), queries with `?` placeholders + `selectionArgs`.

## Intent / IPC
- `PendingIntent` without `FLAG_IMMUTABLE` (API 23+) → hijack.
- Implicit intent carrying sensitive data; `startActivity(getIntent().getParcelableExtra(...))` → intent redirection.
- Reading `getIntent().getStringExtra(...)` and passing it to a sensitive sink without validation.
```bash
grep -rnE 'PendingIntent\.(getActivity|getService|getBroadcast)|FLAG_MUTABLE|getIntent\(\)\.(get|getParcelableExtra)|sendBroadcast\(' --include=*.kt --include=*.java .
```

## Embedded secrets
```bash
grep -rnE '(API_KEY|SECRET|TOKEN|PASSWORD|AIza[0-9A-Za-z_-]{20,})' --include=*.kt --include=*.java res/values/strings.xml gradle.properties local.properties build.gradle* 2>/dev/null
```
Note: BuildConfig fields / `strings.xml` / a committed `local.properties` all ship or leak. Injecting at build time from a CI secret (not committed) → ok.
