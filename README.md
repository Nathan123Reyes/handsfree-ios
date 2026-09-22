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

## Roadmap / not yet built

- **Paywall**: 3-day free trial capped at 5 uses/day, then a monthly subscription or one-time lifetime purchase, via RevenueCat + StoreKit. Needs a RevenueCat project + App Store Connect in-app purchase products set up first — nothing here yet.
- **Server-held API key**: once the paywall exists, the app should call a small backend (e.g. a Netlify Function in `handsfree-web`) that holds the Anthropic key server-side and checks the RevenueCat entitlement, instead of the app calling Anthropic directly with a user-supplied key like the free web version does.
