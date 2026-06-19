# Cosmic AI — Deployment Guide

This is the real, deployable version of Cosmic AI: Firebase Auth (Google + Email),
a Cloud Functions backend that calls Claude safely server-side, real Google Play
Billing verification, and Capacitor to wrap it as an Android app.

Follow these steps **in order**. Each one builds on the last.

---

## Part 1 — Firebase project setup (~20 min)

1. Go to https://console.firebase.google.com → **Add project** → name it (e.g. "cosmic-ai").
2. Once created, click the **web icon (`</>`)** to register a web app. Copy the
   `firebaseConfig` object it gives you.
3. Paste those values into `public/firebase-config.js`, replacing the
   `REPLACE_WITH_...` placeholders.
4. In the Firebase console sidebar: **Build → Authentication → Get started**.
   - Enable **Email/Password** sign-in method.
   - Enable **Google** sign-in method (it auto-fills support email).
5. In the sidebar: **Build → Firestore Database → Create database**. Start in
   production mode (the rules file in this project already locks it down).
6. Install the Firebase CLI on your computer:
   ```
   npm install -g firebase-tools
   firebase login
   ```
7. From inside this project folder:
   ```
   firebase use --add
   ```
   and select the project you just created.

---

## Part 2 — Get your Claude API key

1. Go to https://console.anthropic.com → **API Keys** → create a new key.
2. Don't paste it into any file. You'll store it as a Firebase **secret** in Part 3
   — that's what keeps it out of your APK.

---

## Part 3 — Deploy the backend (Cloud Functions)

```
cd functions
npm install
cd ..

firebase functions:secrets:set ANTHROPIC_API_KEY
# paste your Claude API key when prompted

firebase deploy --only functions,firestore:rules
```

When it finishes, the CLI prints URLs like:
```
chat: https://us-central1-cosmic-ai-xxxxx.cloudfunctions.net/chat
getStatus: https://us-central1-cosmic-ai-xxxxx.cloudfunctions.net/getStatus
verifyPurchase: https://us-central1-cosmic-ai-xxxxx.cloudfunctions.net/verifyPurchase
```

Copy the base part (`https://us-central1-cosmic-ai-xxxxx.cloudfunctions.net`) into
`FUNCTIONS_BASE_URL` in `public/firebase-config.js`.

**Test it works:** deploy hosting too and open it in a browser first, before
touching Android at all:
```
firebase deploy --only hosting
```
Open the URL it gives you, sign up with email, pick Free, and ask a science
question. If that works end-to-end, the hard part is done.

---

## Part 4 — Google Play Console setup (~30 min + $25)

1. Go to https://play.google.com/console → pay the one-time $25 registration fee.
2. **Create app** → fill in name ("Cosmic AI"), category, free app.
3. Go to **Monetize → Products → Subscriptions** → create a subscription:
   - Product ID: `cosmic_ai_premium_monthly` (must exactly match the
     `PREMIUM_PRODUCT_ID` constant in `functions/index.js` and
     `public/firebase-config.js`)
   - Price: $9/month (or whatever you choose)
4. Go to **Setup → API access** → link a Google Cloud project (Firebase already
   created one — link that same one) → this lets your backend verify purchases.
5. Under that same API access page, create a **service account** with the
   "Service Account User" role, then in Google Cloud Console grant it access to
   Play Console with **Finance** permissions (view financial data) at minimum.
6. Download that service account's JSON key file, then store it as a secret:
   ```
   firebase functions:secrets:set PLAY_SERVICE_ACCOUNT_JSON
   # paste the ENTIRE contents of the downloaded JSON file when prompted
   ```
7. Redeploy functions so they pick up the new secret:
   ```
   firebase deploy --only functions
   ```

This step is the fiddliest part of the whole process — Google's Play Console UI
changes occasionally. If a screen doesn't match exactly, search "link Play
Console to Google Cloud service account" for current screenshots.

---

## Part 5 — Wrap as an Android app with Capacitor

```
npm install
npm run android:add
npm run android:sync
npm run android:open
```

The last command opens **Android Studio** (install it first if you haven't:
https://developer.android.com/studio — this needs a real computer, won't run
in a cloud sandbox).

In Android Studio: connect a phone (USB debugging on) or use the built-in
emulator, then press **Run**. You now have Cosmic AI running as a real Android
app, hitting your real backend.

### Google Sign-In on Android (important config step)
`signInWithPopup` (used in `index.html`) doesn't work well inside Capacitor's
WebView. Two options:
- **Easiest**: switch to `auth.signInWithRedirect(provider)` instead — works
  in WebViews with no extra setup, slightly different UX (full page redirect).
- **Best native UX**: use the `@capacitor-firebase/authentication` plugin,
  which calls native Android Google Sign-In and feeds the credential into
  Firebase. More setup (needs your Android app's SHA-1 fingerprint registered
  in Firebase console → Project Settings → Your apps), but feels like a real
  native login button. Search "capacitor-firebase authentication google
  android setup" for the current guide.

### Wiring up Play Billing (for real payments)
The frontend has a hook point ready: `window.CosmicBilling.purchasePremium()`.
Install a Play Billing plugin for Capacitor (e.g. search npm for
"capacitor-purchases" or "cordova-plugin-purchase" — the Capacitor billing
plugin ecosystem changes, so check for one actively maintained). Implement
`window.CosmicBilling.purchasePremium()` to:
1. Launch the real Play Billing purchase flow for `cosmic_ai_premium_monthly`
2. On success, return `{ purchaseToken, productId }`
The app already sends that to `verifyPurchase` on your backend, which checks
it against Google's servers and marks the user premium — this part doesn't
need more code from you, just the plugin wiring.

---

## Part 6 — Publish

1. In Android Studio: **Build → Generate Signed Bundle/APK** → choose
   **Android App Bundle (.aab)** → create a new keystore (back this up
   somewhere safe — losing it means you can never update the app again).
2. In Play Console: **Production → Create release** → upload the `.aab`.
3. Fill out: privacy policy URL (required — even a simple hosted page works),
   content rating questionnaire, app screenshots, store listing description.
4. Submit for review. Google typically takes a few days; expect at least one
   round of feedback if anything's missing.

---

## Costs to expect

- Firebase: free tier covers a small app's auth + Firestore + light function
  usage; you pay only past those limits.
- Claude API: pay-per-token, billed by Anthropic based on actual usage —
  check current pricing at https://www.anthropic.com/pricing before launch,
  since this is your main variable cost per user message.
- Google Play: $25 one-time.
- Google takes a cut of subscription revenue (currently 15% for most
  developers under $1M/year — confirm current terms in Play Console).

## Security notes already built in

- Claude API key lives only in Firebase Secrets, never in the app bundle.
- Firestore rules block users from writing their own `plan` field — only
  Cloud Functions (using the Admin SDK) can grant premium, and only after
  verifying a real purchase with Google's servers.
- Daily free-tier counts are tracked server-side in Firestore, not in the
  app's local state, so they can't be reset by clearing app data.
