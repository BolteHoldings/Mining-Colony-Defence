# Ads + In-App Purchase Integration

The game ships with `Platform` stubs (search `PLATFORM LAYER` in `www/index.html`). This guide replaces them with real AdMob and Play Billing calls.

**Key constraint:** `index.html` is a single file with no bundler, so `import {AdMob} from '@capacitor-community/admob'` will NOT work. Capacitor registers native plugins on the global bridge, so use `window.Capacitor.Plugins.AdMob` instead. Everything below uses that.

---

## Part 1 — AdMob

### 1.1 Console setup
1. AdMob account → **Apps → Add app** → Android → link to your Play listing (or "not published yet").
2. Note the **App ID**: `ca-app-pub-XXXXXXXX~YYYYYYYY` (tilde).
3. Create two ad units under that app:
   - **Rewarded** → for the supply drop. Note its ID (`ca-app-pub-XXXX/ZZZZ`, slash).
   - **Interstitial** → for map transitions. Note its ID.

### 1.2 Install
```bash
npm install @capacitor-community/admob
npx cap sync android
```

### 1.3 Android config
`android/app/src/main/AndroidManifest.xml`, inside `<application>`:
```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXXXXXX~YYYYYYYY"/>
```
(Use your **App ID** — the one with the tilde. Wrong value here = crash on launch.)

### 1.4 Code — replace the `Platform` object in `www/index.html`

```js
// ---- REAL AD IDS ----
const AD_UNITS = {
  rewarded:     'ca-app-pub-XXXXXXXX/1111111111',
  interstitial: 'ca-app-pub-XXXXXXXX/2222222222',
};
const AD_TESTING = false;   // true while developing — NEVER true in a release build

const AdMob = window.Capacitor?.Plugins?.AdMob;
let admobReady = false;

async function initAds(){
  if(!AdMob) return;                       // running in a browser, not the app
  try{
    await AdMob.initialize({ initializeForTesting: AD_TESTING });
    admobReady = true;
    prepareRewarded();                     // preload so the button is instant
  }catch(e){ console.warn('AdMob init failed', e); }
}
function adOpts(unit){
  return { adId: unit, isTesting: AD_TESTING,
           npa: adConsent === 'nonpersonalized' };   // honours the consent screen
}
async function prepareRewarded(){
  if(!admobReady || adsRemoved) return;
  try{ await AdMob.prepareRewardVideoAd(adOpts(AD_UNITS.rewarded)); }catch(e){}
}

const Platform = {
  simulated: false,

  async showRewarded(onReward){
    if(adsRemoved){ onReward(); return; }            // paid users skip the ad, keep the reward
    if(!admobReady){ onReward(); return; }           // fail-open: never withhold a reward on error
    try{
      const item = await AdMob.showRewardVideoAd();  // resolves with the reward when earned
      if(item) onReward();
      prepareRewarded();                             // preload the next one
    }catch(e){
      console.warn('rewarded failed', e);
      onReward();                                    // don't punish the player for an ad failure
      prepareRewarded();
    }
  },

  async showInterstitial(){
    if(adsRemoved || !admobReady) return;
    try{
      await AdMob.prepareInterstitial(adOpts(AD_UNITS.interstitial));
      await AdMob.showInterstitial();
    }catch(e){ console.warn('interstitial failed', e); }
  },

  async purchaseRemoveAds(onDone){ /* see Part 2 */ },
};
```

Then call `initAds();` once at boot — put it next to `loadGame();` near the bottom of the file.

**Also delete the simulated ad screen** once this works: remove the `showAdSim` function and the `<div id="adsim">` element.

### 1.5 Testing
- Set `AD_TESTING = true` and use **Google's test unit IDs** while developing. Tapping your own live ads gets your AdMob account banned.
- Rewarded test unit: `ca-app-pub-3940256099942544/5224354917`
- Interstitial test unit: `ca-app-pub-3940256099942544/1033173712`
- Add your device as a test device in AdMob before ever running with real IDs.

---

## Part 2 — Remove Ads ($1.99)

### 2.1 Play Console
1. **Monetize → Products → In-app products → Create product**
2. Product ID: `remove_ads` (permanent, can't be reused once created)
3. Name/description, price $1.99, **Activate** it.
4. You must have uploaded a build containing the billing library to at least a test track before products go live.
5. **Setup → License testing**: add your Gmail so you can buy it without being charged.

### 2.2 Plugin choice
Two solid options:
- **`@capgo/capacitor-native-purchases`** — free, no backend, Play Billing 7.x. Good fit: you have one non-consumable and no server.
- **RevenueCat (`@revenuecat/purchases-capacitor`)** — free tier, handles receipts/restore server-side. More setup, more robust if you add products later.

Below uses the CapGo plugin (simplest for a single unlock).

```bash
npm install @capgo/capacitor-native-purchases
npx cap sync android
```

Add to `android/app/src/main/AndroidManifest.xml` (usually automatic, verify it's there):
```xml
<uses-permission android:name="com.android.vending.BILLING" />
```

### 2.3 Code

```js
const IAP = window.Capacitor?.Plugins?.NativePurchases;
const REMOVE_ADS_ID = 'remove_ads';

// call at boot, after loadGame()
async function initIAP(){
  if(!IAP) return;
  try{
    // restore: if they already own it (reinstall/new device), unlock silently
    const { transactions } = await IAP.restorePurchases();
    if(transactions?.some(t => t.productIdentifier === REMOVE_ADS_ID)){
      adsRemoved = true; saveGame(); refreshIapRow();
    }
  }catch(e){ console.warn('restore failed', e); }
}

// inside Platform:
async purchaseRemoveAds(onDone){
  if(!IAP){ toast('Purchases unavailable'); return; }
  try{
    const res = await IAP.purchaseProduct({
      productIdentifier: REMOVE_ADS_ID,
      productType: 'inapp',          // non-consumable one-time product
    });
    if(res?.transactionId){
      adsRemoved = true; saveGame();
      Analytics.track('iap_remove_ads');
      onDone && onDone();
    }
  }catch(e){
    console.warn('purchase failed', e);
    toast('Purchase cancelled');
  }
},
```

**Verify the exact method/param names against the installed plugin's README** — these APIs shift between versions, and a wrong param fails silently at runtime.

### 2.4 Required: a Restore Purchases button
Google requires users be able to restore a non-consumable on a new device. `initIAP()` does it automatically at boot, but add a manual button in the shop next to the IAP row calling the same restore logic — reviewers look for it.

---

## Part 3 — Analytics (optional)

Every event already funnels through one function. Add Firebase to the Android project, install a Firebase Analytics plugin, then:

```js
const FB = window.Capacitor?.Plugins?.FirebaseAnalytics;
const Analytics = { q:[], track(ev,props){
  try{ FB && FB.logEvent({ name: ev, params: props || {} }); }catch(e){}
} };
```

---

## Pre-release checklist
- [ ] `AD_TESTING = false` and real ad unit IDs
- [ ] AdMob **App ID** in AndroidManifest (tilde version)
- [ ] `DEV_MODE = false` in index.html (hides the map-unlock test toggle)
- [ ] Purchased `remove_ads` once with a license-test account, confirmed ads stop
- [ ] Uninstall/reinstall → restore returns the purchase
- [ ] Consent screen appears on first launch only; `npa` flag reaches AdMob
- [ ] Data safety form declares AdMob (device/ad ID) + any analytics
- [ ] Privacy policy hosted, placeholders filled, URL in the listing
- [ ] Licensed music swapped in (the synth track is a placeholder for *music* only — SFX are final)
