# Mining Colony Defense — Android Packaging Guide

You have: the finished HTML5 game (`www/index.html`), this Capacitor project, and a Google Play developer account. This takes you from here to a Play Store listing.

## 1. Local setup (one-time)
1. Install **Node.js** (LTS) and **Android Studio** (which installs the Android SDK).
2. In this project folder:
   ```bash
   npm install
   npx cap add android
   npx cap sync android
   npx cap open android      # opens the project in Android Studio
   ```
3. In Android Studio, pick a device/emulator and press Run. The game should boot full-screen. Persistence (saves) works automatically now.

Before building, edit `capacitor.config.json` → set `appId` to your own reverse-domain (e.g. `com.haileygames.miningcolonydefense`). This is permanent once published — choose carefully.

## 2. On-device pass
Play on a real phone and check:
- Touch targets (tower buttons, inspector) are comfortable.
- Performance at 4× speed in late waves (particles + audio). If it stutters on a low-end device, tell Claude — the particle counts are easy to dial down.
- The canvas letterboxes correctly in both orientations. Consider locking landscape in `android/app/src/main/AndroidManifest.xml`: add `android:screenOrientation="landscape"` to the activity.

## 3. Ads (AdMob)
1. Create an AdMob account, register the app, create ad units: one **Rewarded**, one **Interstitial**.
2. Install a Capacitor AdMob plugin (e.g. `@capacitor-community/admob`), then in `www/index.html` replace the bodies of:
   - `Platform.showRewarded(onReward)` → load/show rewarded ad; call `onReward()` in the reward callback.
   - `Platform.showInterstitial()` → show interstitial.
3. The consent screen already stores `adConsent` (`'personalized'` / `'nonpersonalized'`) — pass that to the AdMob request configuration (`npa=1` for non-personalized).
4. Test with AdMob **test unit IDs** first; real IDs only in the release build.

## 4. Remove Ads IAP ($1.99)
1. In Play Console: Monetize → Products → In-app products → create `remove_ads` ($1.99, managed/non-consumable).
2. Install a billing plugin (e.g. `@squareetlabs/capacitor-subscriptions` or RevenueCat's Capacitor SDK — RevenueCat is the least painful).
3. Replace `Platform.purchaseRemoveAds(onDone)` with the purchase flow; on success it already sets `adsRemoved=true` and saves. Also add a **Restore purchases** call on app start (query owned products → set `adsRemoved`).

## 5. Analytics (optional but recommended)
Add Firebase to the Android project (google-services.json), install a Firebase Analytics plugin, and forward events inside `Analytics.track(ev, props)` — every event in the game already routes through that one function.

## 6. Release build
1. Android Studio → Build → **Generate Signed Bundle** → create a keystore (BACK IT UP — losing it means you can never update the app).
2. Build an **.aab** (App Bundle), release variant.

## 7. Play Console submission
- App icon 512×512, feature graphic 1024×500, 4–8 phone screenshots (grab them from your device).
- Short + full description. Lead with the hook: *"Tower defense where your economy is on the map — mine ore, wire it into your towers, and defend the network that powers them."*
- **Privacy policy URL**: host `privacy-policy.html` (fill its [[PLACEHOLDER]] fields first — your name/email/date/AdMob). Any static host works.
- Content rating questionnaire, Data safety form (declare AdMob + Firebase data collection), target audience (13+ recommended since ads are personalized).
- Countries, pricing (free), then submit for review. First review typically takes a few days; you'll likely need 20 testers for 14 days first if your account is new (Play's closed-testing requirement for new personal accounts — check your console, this rule doesn't apply to older accounts).

## 8. Before you hit publish
- Swap the synthesized music for a licensed/royalty-free track (the synth is a placeholder; swap happens in the `startMusic`/`scheduleStep` functions or replace with an `<audio>` loop).
- Turn OFF the map-select test mode default: in `www/index.html`, `let devUnlockAll = true` → `false`.
- Delete or hide the dev unlock button for release if you prefer: search `devUnlock`.
