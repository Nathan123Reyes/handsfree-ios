# Handsfree — iOS (Capacitor wrapper)

Thin native shell so **Handsfree** (the camera math solver — see the sibling `handsfree-web` repo) can ship on the App Store. It's a Capacitor app that loads the live deployed site at `https://hands-free.netlify.app` (configured in `capacitor.config.json` under `server.url`) rather than bundling a copy, so web updates ship instantly without an App Store review.

The native `ios/` folder isn't committed (see `.gitignore`) — it's generated on demand by `npx cap add ios`, which is exactly what `codemagic.yaml` does on the first build. That keeps this repo small and avoids checking in Xcode-generated files that go stale.

## Building (no local Mac needed)

This is set up for **Codemagic**, since development is happening on Windows:

1. Connect this repo in Codemagic and select the `ios-handsfree` workflow (defined in `codemagic.yaml`).
2. In the Codemagic UI, add an **App Store Connect API key** integration (Codemagic docs: Team settings → Integrations → App Store Connect) — this is what `xcode-project use-profiles` and the `app_store_connect` publishing step use for code signing and TestFlight upload. Do this in Codemagic's dashboard, not in this repo, since it's a secret.
3. Set the bundle identifier `com.reyes.handsfree` up in App Store Connect (create the App ID + app record) before the first signed build.
4. Push to trigger a build, or run it manually from the Codemagic dashboard.

## Local iOS dev (only if you get access to a Mac)

```
npm install
npx cap add ios      # generates the ios/ folder (git-ignored)
npx cap sync ios
npx cap open ios      # opens Xcode
```

## Paywall (built, needs your accounts wired up)

The trial + subscription logic is implemented in `handsfree-web`'s `index.html` (shared with the free web app, but gated behind a native-platform check so web visitors never see it) and in `netlify/functions/solve.js` (the server-held-key proxy). None of it can go live until you do the account-side setup below — none of it needs code changes, all of it happens in dashboards:

1. **RevenueCat**: create a project at [app.revenuecat.com](https://app.revenuecat.com). Add an iOS app with bundle ID `com.reyes.handsfree`. Create an entitlement called exactly `pro` (the code checks this identifier). Create your products (a monthly auto-renewable subscription and/or a non-consumable lifetime purchase) and attach both to the `pro` entitlement. Group them into an Offering marked "current" — that's what `Purchases.getOfferings()` in the app reads to show the paywall's plan list.
2. **App Store Connect**: create the matching in-app purchase products there first (same product identifiers you used in RevenueCat) — App Store Connect is the source of truth for products, RevenueCat just reads them. Needs your Apple Developer account.
3. **Public SDK key**: RevenueCat gives you a public iOS API key (starts `appl_`) — this is safe to embed client-side. Paste it into `handsfree-web/index.html`, replacing the placeholder `appl_YOUR_REVENUECAT_PUBLIC_SDK_KEY` in the `initNativeApp()` function, then commit and push (auto-deploys via Netlify).
4. **Secret API key** (server-side entitlement check): RevenueCat also gives you a secret API key for server-to-server calls. Add it as an environment variable named `REVENUECAT_SECRET_API_KEY` in the `handsfree-web` Netlify project (Project configuration → Environment variables) — do this yourself in the dashboard, never commit it.
5. `npm install` (or let Codemagic do it) picks up the `@revenuecat/purchases-capacitor` plugin already listed in `package.json`; `npx cap sync ios` links it into the generated Xcode project.

How the gating actually works, so it's not a black box: on first native launch the app starts a 3-day trial capped at 5 solves/day (tracked client-side in `localStorage` — good enough for v1, but note that's not tamper-proof; a determined user could reset it, which is an acceptable v1 tradeoff, not a security boundary). Trial and entitled requests both go through `/.netlify/functions/solve`, which holds the real Anthropic key so the app never asks the visitor for one. Once the trial is exhausted, the app calls RevenueCat's `getCustomerInfo()` client-side to decide whether to show the paywall, but that's only used to decide what UI to show — every non-trial request also gets independently re-checked server-side: `solve.js` calls `GET https://api.revenuecat.com/v1/subscribers/{appUserId}` with the secret key and checks whether the `pro` entitlement is active before it will spend an Anthropic call, so a paying check can't be bypassed just by tampering with the client.

## Still open

- **Device testing**: none of the RevenueCat/paywall code has run on an actual device or simulator (this was written without Xcode access). Test the trial countdown, the paywall's plan list, a real sandbox purchase, and Restore Purchases before shipping.
- **App User ID identity**: RevenueCat assigns an anonymous `appUserID` per install by default, which is what the server check keys off. If you ever want purchases to follow a logged-in user across devices/reinstalls, you'd call `Purchases.logIn(yourUserId)` once you have some notion of an account — not needed for v1 since this app has no login.
