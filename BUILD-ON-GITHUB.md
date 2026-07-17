# Building on GitHub (no Android Studio)

Yes — you can build this entirely in GitHub Actions. `.github/workflows/android.yml` is ready to go.

## What it does
- **Every push to `main`** → builds a **debug APK** you can download and sideload onto your phone.
- **Push a tag like `v1.0.0`** → builds a **signed release AAB** for the Play Console.
- Both land under the **Actions tab → your run → Artifacts**.

## Setup

### 1. Push this project to a GitHub repo
```bash
git init
git add .
git commit -m "Mining Colony Defense"
git branch -M main
git remote add origin https://github.com/YOU/mining-colony-defense.git
git push -u origin main
```
Watch the Actions tab — the debug APK appears in a few minutes. Download, transfer to your phone, tap to install (allow "install from unknown sources"). That's your device test.

### 2. Create a keystore (one-time, needed only for release)
This needs `keytool`, which ships with **any JDK** — no Android Studio. If you published an app before, **reuse that keystore if it's the same app**; otherwise make one:
```bash
keytool -genkey -v -keystore release.keystore -alias mcd \
  -keyalg RSA -keysize 2048 -validity 10000
```
**Back this file up somewhere permanent.** Lose it and you can never update the app on Play — you'd have to publish a new listing and lose your installs and reviews.

### 3. Add repo secrets
Repo → Settings → Secrets and variables → Actions → New repository secret:

| Secret | Value |
|---|---|
| `KEYSTORE_BASE64` | output of `base64 -w0 release.keystore` (macOS: `base64 -i release.keystore`) |
| `KEYSTORE_PASSWORD` | the store password you chose |
| `KEY_ALIAS` | `mcd` (or whatever you used) |
| `KEY_PASSWORD` | the key password |

### 4. Release
```bash
git tag v1.0.0
git push origin v1.0.0
```
Download the AAB artifact → upload to Play Console.

---

## Important: commit the `android/` folder once you configure ads

The workflow runs `npx cap add android` only if `android/` doesn't exist. The moment you edit native files — and **you will**, for the AdMob App ID in `AndroidManifest.xml` — you must commit that folder or CI will regenerate it and silently wipe your changes:

```bash
npx cap add android      # run once, locally, needs Node only
# edit android/app/src/main/AndroidManifest.xml → add AdMob App ID
git add android
git commit -m "Add android platform + AdMob config"
```
If you don't want to run anything locally, the alternative is a workflow step that patches the manifest with `sed` after `cap add` — messier, but it works.

## What you give up without Android Studio
- **No emulator and no logcat.** If the app crashes on launch, you get no stack trace — just a black screen. This is the real cost. `chrome://inspect` on a connected phone gets you the JS console back, which covers most of it since your game is HTML/JS.
- **Slower loop.** Push → wait ~3 min → download → sideload, versus pressing Run.

For a single-file HTML game this is a fine tradeoff. If you hit a native crash you can't diagnose, that's the moment to install Android Studio.

## Also possible: skip the store entirely
It's a web game. `www/index.html` works on any static host — GitHub Pages included (repo Settings → Pages → serve from `/www`). Saves work, ads and IAP are the tradeoff. Some devs ship both.
