# Device-test fallback attempts — 2026-10-03

Target APK: `app/build/outputs/apk/debug/app-debug.apk` (62MB, V2-signed, arm64-v8a + armeabi-v7a present — see `static-audit-2026-10-03.txt`, PASSES static audit)

## Path 1: BrowserStack App Automate free trial
Status: ✅ COMPLETE (2026-10-03 ~21:27 AST)
- Account: "Vasily Starostenko" <vasilyespana@gmail.com>, free plan 100 mins, no card.
- Device: Google Pixel 7, Android 13.0 — REAL physical device (App Live).
- INSTALL: SUCCESS — app-debug.apk uploaded, parsed as v1.0, installed automatically at session start. Logcat shows me.callpages.merchant process running.
- LAUNCH: SUCCESS — first screen renders: "CallPages Merchant" / "Manage your calls, pages and callbacks." / blue "Sign in with Google" button. No crash, no error dialogs.
- Warnings only (non-fatal): Jetpack Compose "failed lock verification and will run slower", FirebaseApp not initialized (expected — no google-services.json yet).
- Evidence: `app-launched.png` (screenshot of app running on the Pixel 7).
- Caveats: (1) debug build with debug key, not release-signed; (2) 1 physical device tested, not 2; (3) interactive App Live session, not an automated Robo crawl.
NOTE: account password was browser-generated; must relay to Vasily once and advise him to store/rotate it.

## Path 2: Headless ARM container (redroid/Anbox via Docker)
Status: BLOCKED (verified 2026-10-03)
- `docker` not installed (`command not found`)
- `/dev/binder*` and `/dev/ashmem` absent — sandbox kernel cannot run Android containers even if Docker were installed
- Sandbox is an unprivileged container (no KVM, no nested virt) — dockerd could not run containers anyway
- Anbox needs the same binder/ashmem kernel modules — equally blocked

## Path 3: AWS Device Farm trial CLI
Status: BLOCKED (verified 2026-10-03)
- No AWS credentials in environment, no `~/.aws`, no AWS CLI config
- AWS account creation requires email verification + credit card — cannot be done autonomously

## Local emulator (bonus path, attempted before fallback directive)
Status: BLOCKED (verified 2026-10-03)
- x86_64 AVD: "x86_64 emulation currently requires hardware acceleration" — no KVM (/dev/kvm absent, no svm/vmx CPU flags)
- arm64 AVD: emulator 37.x FATAL — "Avd's CPU Architecture 'arm64' is not supported by the QEMU2 emulator on x86_64 host"

## Release-signed APK (2026-10-03 ~22:06 AST)
- Keystore: `release.keystore` (RSA-2048, 30-year validity, PKCS12; gitignored; passwords in `keystore.properties`, chmod 600, gitignored)
- Build: `./gradlew assembleRelease` — BUILD SUCCESSFUL (AGP upgraded 8.5.2 -> 8.7.3 + Gradle 8.7 -> 8.9 to fix R8 8.5.35 ConcurrentModificationException; fixed WorkManagerInitializer lint error; PKCS12 requires keyPassword == storePassword)
- R8: minifyEnabled=true, shrinkResources=true, proguard-android-optimize.txt — full obfuscation pipeline green
- Output: `app-release.apk`, 52.8MB (vs 62MB debug — R8 shrinking effective)
- Static audit: `release-audit-2026-10-03.txt` — apksigner Verifies (V2), signer DN CN=CallPages Merchant (release key, NOT debug), arm64-v8a+armeabi-v7a, minSdk 26, targetSdk 34, MainActivity launchable
- BrowserStack release install/launch test: IN PROGRESS (Pixel 7 real device)

## CI gate (2026-10-03)
- Workflow: `.github/workflows/android-device-gate.yml` + `.github/scripts/bs_device_gate.py` — committed locally (YAML + Python validated)
- Gate: every PR to main builds release APK -> apksigner/aapt2 static check -> BrowserStack upload -> Appium install+launch on real Pixel 7 -> screenshot artifact
- BLOCKED on repo creation: fine-grained PAT lacks createRepository scope. Local git repo ready (2 commits); needs `vasilyespana/callpages-android` (private) created, then: push, `gh secret set` x6 (BROWSERSTACK_USERNAME, BROWSERSTACK_ACCESS_KEY, ANDROID_KEYSTORE_BASE64, KEYSTORE_PASSWORD, KEY_ALIAS, KEY_PASSWORD), branch protection requiring the device-gate check.

## Release device test (2026-10-03 ~22:09 AST) — ✅ PASSED
- Device: Google Pixel 7 Pro, Android 13.0 — REAL physical device (App Live; base Pixel 7 was busy)
- INSTALL: SUCCESS — app-release.apk uploaded, parsed, installed automatically at session start. Logcat shows me.callpages.merchant (pid 27098) running.
- LAUNCH: SUCCESS — first screen renders: "CallPages Merchant" / "Manage your calls, pages and callbacks." / blue "Sign in with Google". PROVES R8 obfuscation did not break startup.
- Errors: none. Info warnings only: no jdwp debugger (normal for release), FirebaseApp not initialized (expected — no google-services.json).
- Evidence: `app-release-launched.png`
