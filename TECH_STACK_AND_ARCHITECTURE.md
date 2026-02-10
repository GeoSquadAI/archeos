# LifePods — Tech Stack, Architecture & RevenueCat

A concise reference for the technology choices, app structure, and in-app purchase implementation.

---

## 1. Tech Stack

**Platform & language**
- **Flutter** (Dart SDK ≥3.0) — cross-platform mobile (Android, iOS) and web
- Targets: Android, iOS, Web, Windows (desktop)

**Core dependencies**

| Category | Packages | Purpose |
|----------|----------|---------|
| **State** | `provider` ^6.1.1 | App-wide state (commute, learning, profile, RevenueCat) |
| **Navigation** | `go_router` ^12.1.1, `MaterialApp.routes` | Named routes; bottom nav (Home, Explore, Commute, Library, Profile) |
| **Maps & location** | `google_maps_flutter`, `google_places_flutter`, `flutter_google_places_sdk`, `geolocator`, `geocoding` | Directions, place autocomplete, current location |
| **Media** | `just_audio`, `audio_service`, `video_player` | Audio/video (player UI currently simulated) |
| **Networking** | `http` | API calls (e.g. Google Directions, future AI) |
| **Storage** | `shared_preferences` | Local persistence (available; not yet used for quotes/library) |
| **Config** | `flutter_dotenv` | `.env` for `GOOGLE_MAPS_API_KEY` (and other secrets) |
| **UI** | `google_fonts`, `cupertino_icons`, `confetti` | Typography, icons, premium celebration |
| **In-app purchases** | `purchases_flutter` ^9.11.0 | RevenueCat SDK for subscriptions |

**Backend / external services**
- **Google**: Maps (Directions, Places), optional sign-in
- **RevenueCat**: Subscription management, receipt validation, entitlements
- **AI** (planned): `AIPodcastService` is stubbed; will call OpenAI/Claude or custom endpoint when wired

---

## 2. Architecture

**High-level flow**
- **Entry**: `main.dart` loads `.env`, initializes RevenueCat, registers providers, and runs `MyApp` with `MaterialApp` and named routes.
- **Screens** drive UI; **providers** hold state; **services** perform I/O (maps, AI, purchases).

**Directory layout**

```
lib/
├── main.dart                 # Bootstrap, providers, routes, theme
├── models/                   # place, podcast, user_preferences
├── providers/                # CommuteProvider, LearningProvider, ProfileProvider
├── screens/                  # Welcome, CommuteSetup, CreatePodcast, Player, Explore, Library, Profile, Premium
├── services/                 # RevenueCat, AI podcast, Google Maps (mobile/web)
├── theme/                    # app_theme.dart
├── utils/                    # constants
└── widgets/                  # Bottom nav, modals, place autocomplete, etc.
```

**State management**
- **Provider**: `ChangeNotifierProvider` for `CommuteProvider`, `LearningProvider`, `ProfileProvider`, and `RevenueCatService`.
- Screens use `Provider.of<T>()` or `Consumer<T>` to read/update state and react to changes (e.g. premium status, commute time, playback progress).

**Routing**
- Routes are defined in `main.dart`: `/`, `/commute-setup`, `/create`, `/podcast-player`, `/explore`, `/library`, `/profile`, `/premium`.
- Navigation: `Navigator.pushNamed` / `pushReplacementNamed`; bottom bar switches between main tabs.

**Data flow (typical flows)**
- **Commute → Podcast**: CommuteSetup → `CommuteProvider.calculateCommute()` (Google Directions) → commute time modal → Create Podcast → `AIPodcastService.generatePodcast()` (currently mock) → Podcast Player.
- **Premium**: `RevenueCatService` is initialized at startup; Premium screen and any paywall use `revenueCat.isPremium` and call `purchaseMonthly()`, `purchaseAnnual()`, or `restorePurchases()`.

**Platform-specific services**
- `GoogleMapsService` has mobile and web implementations (`google_maps_service_mobile.dart`, `google_maps_service_web.dart`); Maps API key comes from `.env` (and in web, from `index.html` until deployment is locked down).
- RevenueCat is initialized only on Android and iOS; web/desktop skip purchase setup.

---

## 3. RevenueCat Implementation

**Role**
- RevenueCat provides a single API for subscription management, receipt validation, and entitlement checks across stores (Google Play, App Store when added).
- The app uses one entitlement, **premium**, to gate premium features (unlimited commute pods, mood selection, theme blending, Gabby voice, Surprise Me, priority access).

**Components**

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **Config** | `lib/services/revenuecat_service.dart` → `RevenueCatConfig` | API keys (Android/iOS), entitlement ID `premium`, product IDs for monthly/annual |
| **Service** | `RevenueCatService` (singleton, `ChangeNotifier`) | Init SDK, listen for customer info, expose `isPremium`, load offerings, purchase/restore, identify/reset user |
| **UI** | `lib/screens/premium_screen.dart` | Paywall; monthly/annual purchase buttons; restore; “You’re Premium!” when `revenueCat.isPremium` |
| **Bootstrap** | `lib/main.dart` | `await RevenueCatService().initialize()` before `runApp`; `ChangeNotifierProvider.value(value: RevenueCatService())` |

**Initialization**
- In `main()`, after `WidgetsFlutterBinding.ensureInitialized()` and `dotenv.load()`:
  - `RevenueCatService().initialize()` configures the SDK with the platform-specific API key, sets debug log level in debug builds, adds a customer-info listener, then calls `refreshCustomerInfo()` and `loadProducts()`.
- On unsupported platforms (e.g. web), init returns early without configuring Purchases.

**Product and entitlement model**
- **Entitlement**: `premium` — active when the user has an active subscription linked to this entitlement in RevenueCat.
- **Products** (must match Google Play Console and RevenueCat offerings):
  - `lifepods_premium_monthly` — monthly subscription (e.g. $4.99, 7-day trial).
  - `lifepods_premium_annual` — annual subscription (e.g. $39.99).
- Offerings are fetched via `Purchases.getOfferings()`; the “current” offering’s packages are used for purchase and for displaying prices.

**Purchase flow**
- User taps Monthly or Annual on `PremiumScreen` → `RevenueCatService.purchaseMonthly()` or `purchaseAnnual()` → `purchasePackage(productId)`.
- Service resolves the package from current offerings, calls `Purchases.purchasePackage()`, then updates internal state from the returned `CustomerInfo`. `_updateCustomerInfo()` sets `_isPremium` from `customerInfo.entitlements.active.containsKey('premium')` and calls `notifyListeners()` so the UI updates.
- Result is returned as `RcPurchaseOutcome` (success, cancelled, or error message). Restore uses `Purchases.restorePurchases()` and returns `RcRestoreOutcome`.

**Checking premium in the app**
- **In widgets**: `Consumer<RevenueCatService>` and branch on `revenueCat.isPremium` (e.g. show paywall vs “You’re Premium!”).
- **In code**: `Provider.of<RevenueCatService>(context, listen: false).isPremium` or direct service access for non-UI logic.
- **User identity**: Optional `identifyUser(userId)` on login and `resetUser()` on logout for cross-device entitlement sync.

**Error handling**
- Purchase errors (e.g. cancelled, not allowed, product not available, network, store problem) are mapped to user-facing strings in `_getReadableError()`. Cancelled purchases are treated as non-fatal (no error message). All outcomes are reflected in `RcPurchaseOutcome` / `RcRestoreOutcome` and in the service’s `errorMessage` and `notifyListeners()`.

**Configuration checklist**
- Replace `RevenueCatConfig.androidApiKey` and `iosApiKey` with keys from the RevenueCat dashboard.
- In RevenueCat: create entitlement `premium`, create offering (e.g. `default`) with packages tied to `lifepods_premium_monthly` and `lifepods_premium_annual`, and connect Google Play (and later App Store) with the correct app and service account / credentials.
- On Android: `BILLING` permission in `AndroidManifest.xml`; products created and activated in Google Play Console.

For step-by-step setup (accounts, Play Console, RevenueCat dashboard, testing), see **REVENUECAT_SETUP.md**.

---

*Last updated: February 2026*
