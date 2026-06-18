# Android shell app

A ~300-line wrapper that exists for one reason: **Chrome on Android currently
can't reach a wired CDC serial device** — WebUSB fences the CDC-data interface,
and wired Web Serial is still rolling out (device-gated, 2026). Native apps use
the Android USB Host API, which has no such restriction.

It's a fullscreen **WebView** that loads the GitHub Pages app
(`https://bigjosh.github.io/arduino-mosfet-scanner/`) and injects
`window.AndroidSerial`, a USB-serial bridge backed by
[usb-serial-for-android](https://github.com/mik3y/usb-serial-for-android) (CDC,
CH340, FTDI, CP210x — so clone boards work too). The web app's transport layer
auto-prefers the bridge when present. **All UI features ship via the web app** —
this APK only needs rebuilding when the bridge itself changes (or the Pages URL
changes).

Extras the shell adds over a browser: native USB permission flow (plugging the
rig in grants it automatically), keep-screen-on, CSV export into the system
Downloads folder, and no background-tab timer throttling.

## Install (users)

See **[../how-to-install.md](../how-to-install.md)** for the full walkthrough.
Short version: grab `app-release.apk` from
[Releases](https://github.com/bigjosh/arduino-mosfet-scanner/releases), tap it on
the phone, allow your browser to install unknown apps (one-time), Install.
Updates: install a newer APK over the top — history/settings survive (same
signing key).

## Build (maintainers)

Toolchain (no Android Studio needed): JDK 17, Android cmdline-tools with
`platforms;android-34` + `build-tools;34.0.0`, Gradle 8.7. Point
`android/local.properties` at your SDK (`sdk.dir=...`).

```powershell
$env:JAVA_HOME = "<path-to-jdk-17>"
gradle -p android assembleRelease
# -> android/app/build/outputs/apk/release/app-release.apk
```

Signing: `android/release.keystore` + `android/keystore.properties` (both
**git-ignored — back them up**; losing the keystore breaks update-in-place for
installed users). Without them the build falls back to an unsigned release (use
`assembleDebug` for local testing).

**Forking under a different repo name?** The Pages URL is compiled in — update
`APP_URL` in `app/src/main/java/com/bigjosh/mosfetscanner/MainActivity.java` to
your `https://<user>.github.io/<repo>/` and rebuild.

Publish: bump `versionCode` / `versionName` in `app/build.gradle`, then
```powershell
gh release create app-vX.Y.Z android/app/build/outputs/apk/release/app-release.apk
```

## Bridge protocol (JS side)

```
AndroidSerial.list()            -> '[{"id","vid","pid","name","driver"}]'
AndroidSerial.connect(id, baud) -> events via window.__androidSerialEvent
AndroidSerial.write(base64)     -> bool
AndroidSerial.close()
AndroidSerial.saveFile(name, base64)  -> system Downloads + toast
events: {type: 'connect'|'data'|'error'|'disconnect', data}  (data = base64)
```
