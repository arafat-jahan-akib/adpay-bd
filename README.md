# AdPay BD — Android App

A Capacitor wrapper that packages the AdPay BD web prototype (`www/index.html`) into a real installable Android APK.

This is still a **UX prototype** — there is no backend. All data (balances, users, ad campaigns, share submissions) is mock data defined inside `www/index.html`. Wiring it to a real backend, payment gateway, and Facebook verification API is a separate, larger project.

## What's in this repo

```
├── www/index.html              ← the entire app (React + Tailwind, loaded from CDN, single file)
├── capacitor.config.json       ← Capacitor app config (app id, name, web folder)
├── package.json                ← Node dependencies (Capacitor CLI + Android runtime)
├── android-icons/              ← pre-generated app icon at every required Android density
├── icon-source.png             ← the source crop of the AdPay BD logo mark used to generate the icons
├── .github/workflows/build-apk.yml   ← builds a debug APK automatically on every push
└── .gitignore
```

There is no `android/` folder committed — it's generated automatically (by Capacitor) the first time you build, either locally or via the GitHub Action below. This keeps the repo small and avoids committing generated native code.

## Easiest option: let GitHub build the APK for you

You don't need Android Studio or a local build environment for this.

1. Create a new GitHub repository and push these files to it (see "Pushing to GitHub" below).
2. Go to the **Actions** tab in your repo. The `Build Android APK` workflow runs automatically on every push to `main`, or you can trigger it manually with **Run workflow**.
3. When it finishes (a few minutes), open the completed run and download the **`adpay-bd-debug-apk`** artifact — that's your installable `.apk`.
4. Transfer it to an Android phone and tap it to install (you'll need to allow "install from unknown sources" the first time).

This produces a **debug APK**, which is fine for installing and testing on your own devices. It is not signed for the Play Store — see "Release build" below for that.

### Pushing to GitHub for the first time

```bash
git init
git add .
git commit -m "Initial AdPay BD Capacitor project"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

## Building locally instead (optional)

If you'd rather build on your own machine:

**Requirements:** Node.js 20+, Java JDK 17, Android SDK (Android Studio is the easiest way to get this).

```bash
npm install
npx cap add android      # only needed the first time — generates the android/ folder
npx cap sync android
cd android
./gradlew assembleDebug
```

The APK will be at `android/app/build/outputs/apk/debug/app-debug.apk`.

Alternatively, after `npx cap add android`, run `npx cap open android` to open the project in Android Studio and build/run from there — this also lets you run it on an emulator or plug in a phone via USB.

## Changing the app

Everything about the app's UI, screens, and mock data lives in `www/index.html`. After editing it:

```bash
npx cap sync android
```

then rebuild. If you're using the GitHub Action, just commit and push — it re-syncs and rebuilds automatically.

## App identity

- **App ID:** `bd.adpay.app` (set in `capacitor.config.json`) — this is the unique package identifier Android uses. Change it before any real release if you plan to publish, since it can't be changed again after your first Play Store upload.
- **App name:** `AdPay BD`
- **App icon:** the blue-and-gold "A" play-button mark from the AdPay BD logo, on a navy (`#000D2B`) background. Pre-generated at every Android density (mdpi through xxxhdpi) plus a modern adaptive icon (separate foreground/background layers so it displays correctly whether a device masks icons as circles, squircles, or otherwise) — all in `android-icons/`. The GitHub Actions workflow copies these into the generated native project automatically on every build, right after `npx cap add android` creates it — you don't need to do anything manually.

  If you build locally instead of via GitHub Actions, copy them in yourself after running `npx cap add android`:
  ```bash
  RES=android/app/src/main/res
  cp android-icons/values/ic_launcher_background.xml "$RES/values/"
  cp android-icons/mipmap-anydpi-v26/*.xml "$RES/mipmap-anydpi-v26/"
  for d in mdpi hdpi xhdpi xxhdpi xxxhdpi; do
    cp android-icons/mipmap-$d/*.png "$RES/mipmap-$d/"
  done
  npx cap sync android
  ```
  To swap in a different icon later, replace `icon-source.png` (ideally a clean square crop, ~450×450px+) and regenerate the density set, or just edit the PNGs under `android-icons/` directly at each size.

## Release build (signed, for Play Store)

The debug APK from the steps above works for installing on your own devices but is not accepted by the Play Store and shows an "unverified developer" warning on install. To produce a signed release build:

1. Generate a signing keystore:
   ```bash
   keytool -genkey -v -keystore adpay-release.keystore -alias adpay -keyalg RSA -keysize 2048 -validity 10000
   ```
2. Configure signing in `android/app/build.gradle` (a `signingConfigs` block referencing your keystore).
3. Build with `./gradlew assembleRelease` instead of `assembleDebug`.

Keep the keystore file and its passwords safe and private — losing it means you can never update the app under the same listing again. This step is intentionally not automated here since it involves secrets that shouldn't be committed to a repo; ask if you want the GitHub Action extended to handle signed release builds using GitHub Secrets.

## Known limitations of the current build

- **Requires internet on every launch.** The app loads React, Babel, Tailwind, and fonts from CDNs at runtime rather than bundling them — so the phone needs a working connection to open the app at all, not just to sync data. For a real release, these should be bundled locally instead (a follow-up task, not done here).
- No backend — balances, users, campaigns, and Facebook share submissions are all mock/static data in `www/index.html`.
- The Facebook Share task's link-based verification is manual (admin review), not automated — there's no real Facebook API integration.
- bKash withdrawal and SSLCommerz/payment integration points are UI-only stubs.
- Splash screen still uses Capacitor's default — only the app icon has been customized so far.
