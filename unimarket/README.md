# UniMarket

UniMarket is a mobile marketplace built for university students to buy and sell things within their own campus community — old textbooks, electronics, dorm stuff, whatever. It's a Flutter app backed by Firebase, built as the final project for a mobile application development course (ISIS3510, Universidad de los Andes, first semester of 2024) by a two-person team.

This is **not** a production app. It was built over a semester to satisfy course requirements (working auth, a database-backed feature, an offline-tolerant feature, a sensor-based feature, etc.), and the codebase reflects that: some corners were cut to hit deadlines, some features are half-wired, and a few are demo gimmicks added specifically to check a rubric box. The "Lessons learned" section at the bottom is an honest account of what I'd do differently.

## The problem it (tries to) solve

Students end every semester with stuff they don't need anymore — textbooks for classes they'll never retake, calculators, lab equipment, furniture — and the next cohort needs exactly that stuff. Generic marketplaces (Facebook Marketplace, Mercado Libre, campus WhatsApp groups) work, but they mix in strangers from across the city, have no sense of "this is my university," and don't give buyers and sellers who are both on campus a lightweight way to chat and coordinate a handoff. UniMarket narrows the marketplace down to people who already share a campus, with in-app chat so a deal doesn't require exchanging phone numbers with a stranger.

## Features

What actually works in the current build:

- **Email/password auth** (Firebase Authentication) with a sign-up flow that creates a matching user document in Firestore.
- **Product catalog**: browse, search, and filter by category and condition (new vs. second-hand), with a "trending" list computed from view counts.
- **Publish a product**: name, price, description, category, condition, and a photo taken with the camera or picked from the gallery, uploaded to Firebase Storage.
- **Cart**: add products to a per-user cart stored in Firestore.
- **Chat**: create or join topic/group-style chats and exchange messages in real time via Firestore snapshots, with a basic offline queue (messages written while offline are stashed in `SharedPreferences` and replayed once connectivity returns).
- **Profile / user info form**: a short onboarding form (age, name, nickname, university, etc.) persisted locally with `shared_preferences`.
- **Light/dark theme toggle**, persisted across sessions.
- **Connectivity awareness**: a global online/offline stream (`connectivity_plus`) that the UI reacts to — e.g., publishing a product while offline queues it instead of failing outright.
- **Two "sensor" gimmicks** built to satisfy the course's "use a device sensor" requirement: shaking the phone jumps straight to the publish screen (`shake` package), and a background noise meter pops up an alert if ambient noise crosses 90 dB while the app is open.

What's visibly incomplete (see "Lessons learned"): the "Add to cart" button on the product detail screen is wired to an empty function, "Sign Out" in Settings navigates back to the login screen without actually calling `FirebaseAuth.signOut()`, and the seller shown on a product's detail page is a hardcoded placeholder rather than the real seller record.

## Tech stack

- **Flutter / Dart** (Dart SDK ^3.2.6) — the app is generated for Android, iOS, macOS, Web, Linux, and Windows, but it was only ever really built and tested for **Android**; the other platform folders are unused scaffolding left over from `flutter create`.
- **Firebase**
  - Firebase Auth — email/password sign-in
  - Cloud Firestore — products, carts, users, chat groups/messages
  - Firebase Storage — product images
- **State management**: `provider`, used narrowly for theme and connectivity — most app state actually lives in a hand-rolled singleton (see Architecture below), not in the Provider tree.
- **Local persistence**: `shared_preferences` for user profile data, theme choice, and the offline chat message queue.
- Notable packages: `image_picker` / `camera` (product photos), `connectivity_plus` / `internet_connection_checker` (network state), `cached_network_image` (image caching), `shimmer` / `loading_animation_widget` (loading states), `shake` and `noise_meter` (the two sensor features), `flutter_isolate` (background connectivity polling), `uuid`, `url_launcher`, `permission_handler`.

## Architecture overview

The code lives under `lib/` and is organized loosely by responsibility rather than by feature:

```
lib/
├── Controllers/     # ChangeNotifier classes, one per screen-ish concern
├── Models/
│   ├── Repository/  # Direct Firestore/Storage access (one class per collection)
│   └── *.dart        # Plain data classes (ProductModel, UserModel, SellerModel...)
├── Views/            # Screens and widgets, organized by feature folder
├── resources/        # Cross-cutting services (connectivity, shimmer effect)
├── theme.dart         # Light/dark ThemeData + ThemeNotifier
├── globals.dart       # App-wide mutable singleton (session UID, isolate counter)
├── nav_bar.dart       # Bottom navigation bar
├── firebase_options.dart  # Generated FlutterFire config
└── main.dart
```

It's a rough approximation of MVC:

- **Views** are Flutter widgets (mostly `StatefulWidget`) that own their own UI state and call into a Controller or straight into a Repository.
- **Controllers** are thin — several (`HomeController`, `CartController`) barely do more than forward a call to the data layer. They extend `ChangeNotifier` but aren't consistently wired into `Provider`, so most of them don't actually drive rebuilds — they're closer to plain service classes.
- **Repositories** (`ProductRepository`, `CartRepository`, `DatabaseService` for chat) talk to Firestore/Storage directly, with no interface or abstraction over "the backend" — swapping Firestore for anything else would mean touching every repository.
- **`Model`** (`lib/Models/model.dart`) is the odd one out: a singleton that holds the in-memory product catalog, the current cart, and the session's user ID, and every layer reaches into it directly. It's effectively the app's real global state container, in a codebase that otherwise imports Provider for that job. This is the single biggest architectural inconsistency in the project (more in "Lessons learned").
- **`Globals`** (`lib/globals.dart`) is a second, smaller singleton just for the session UID and a counter of running background isolates — a natural candidate to merge with `Model`.

Data flow, roughly: `View` → `Controller` (thin passthrough) → `Model` singleton (in-memory cache) and/or `Repository` (Firestore/Storage) → back up through a `Future`/`Stream` the View awaits or listens to directly.

## Running it locally

**Prerequisites**
- Flutter SDK (this project targets `>=3.2.6 <4.0.0`) — [install guide](https://docs.flutter.dev/get-started/install)
- A physical Android device or emulator (the camera, microphone, and shake-detection features don't do much on desktop/web targets)
- A Firebase project of your own if you want write access — see below

**Steps**

```bash
git clone <this-repo>
cd unimarket
flutter pub get
flutter run
```

That's it for read-only exploration — the repo ships with a working (shared, class-project) Firebase configuration already wired into `lib/firebase_options.dart`, `android/app/google-services.json`, and `ios/Runner/GoogleService-Info.plist`, so `flutter run` against an emulator/device will connect to Firestore and Storage out of the box.

**If you want your own backend** (recommended if you're forking this rather than just poking at it):

1. Create a Firebase project in the [Firebase console](https://console.firebase.google.com/).
2. Enable **Authentication** (Email/Password provider), **Cloud Firestore**, and **Storage**.
3. Install the FlutterFire CLI and regenerate the config for your project:
   ```bash
   dart pub global activate flutterfire_cli
   flutterfire configure
   ```
   This overwrites `lib/firebase_options.dart` and drops fresh `google-services.json` / `GoogleService-Info.plist` files into the platform folders.
4. Firestore needs three collections that the app writes to directly with no schema/migration tooling: `products`, `carts` (keyed by user UID), and `users`/`groups`/`messages` for chat. There's no seed script — the app just creates documents as you use it (sign up creates a `users` doc, first login creates an empty `carts` doc, publishing creates a `products` doc).
5. Firestore/Storage security rules aren't included in this repo — you'll need to write your own before treating this as anything but a local sandbox (see "Lessons learned").

## Environment variables

There aren't any — and that's worth calling out rather than glossing over. Every configuration value the app needs (Firebase API keys, project ID, storage bucket) is baked into `lib/firebase_options.dart` and the platform-specific Firebase config files, all of which are committed to the repo. That was fine for a shared class project where the whole team needed the same backend with zero setup friction, but it's not how I'd do it again:

- Firebase's client-side API keys aren't secret in the way a server API key is — they identify the project, not authenticate requests — so committing them isn't the security hole it looks like at first glance, **provided** Firestore/Storage security rules actually restrict access (ours mostly don't — see below).
- Even so, the right pattern is to keep `firebase_options.dart` and the platform config files out of git (via `flutterfire configure`, run locally by each developer) and commit a `.example` template instead, so the repo doesn't hardcode which backend you're pointed at.
- If this project grew a real backend key (a paid maps API, a payment provider, anything server-side), it would need to go through `--dart-define` / `flutter_dotenv` / a CI secrets store — none of that infrastructure exists here today.

## Lessons learned / what I'd improve

This was my (our) first real Flutter app, built under semester deadlines with two people splitting features, and it shows. In roughly the order I'd tackle them if I revisited this project:

- **The `Model` singleton is a god object, and it's the actual state management layer.** We added `provider` early on for theme and connectivity, but almost everything else — the product catalog, the cart, the session user ID — lives in a mutable singleton that every Controller and several Views reach into directly. It works, but it means nothing is testable in isolation, there's no single source of truth for "what triggers a rebuild," and two different state-management philosophies coexist in the same app for no good reason. I'd pick one pattern (Provider/Riverpod with proper `ChangeNotifier`s, or a lightweight Bloc) and migrate everything onto it.
- **No tests.** `test/widget_test.dart` is still the counter-app test `flutter create` generates by default — it doesn't even reference this app's features. With Firestore calls scattered directly inside Views and Controllers with no interfaces to mock, retrofitting tests now would mean restructuring the data layer first, not just adding test files.
- **Firestore is called directly from four or five different layers with no shared abstraction.** `ProductRepository`, `CartRepository`, and `DatabaseService` each construct their own `FirebaseFirestore.instance` and hand-build documents field-by-field with string keys (`'sellerId'`, `'used'`, etc.) — typos in a field name would fail silently. A shared data layer with typed converters (`withConverter`) would have caught bugs at compile time instead of in Firestore's console.
- **Firestore/Storage security rules were never really finished.** The app happily writes to `products`, `carts`, and `users` from the client with essentially no server-side validation. For a class demo that's a shortcut you take under time pressure; for anything real it's the first thing I'd fix, before touching UI.
- **Error handling is mostly "catch and hope."** `AuthController.registrar` catches exceptions and returns the exception object as if it were a success value; several repository methods have no error handling at all and will just throw on a flaky connection. Combined with `print()` statements left in for debugging instead of real logging, diagnosing a failure in the field would be painful.
- **A few features are visibly unfinished.** "Add to cart" on the product detail screen is bound to an empty function; "Sign Out" navigates back to the login screen without calling `FirebaseAuth.instance.signOut()`, so the Firebase session technically stays alive; the seller info shown on a product page is a hardcoded placeholder (`'Seller name'`, a fake phone number) rather than pulled from the actual seller's user document. These are the kind of things that get left behind when a feature "looks done" in a demo but the last 10% never gets scheduled.
- **The background connectivity check is over-engineered and redundant.** `NetworkController` spawns a whole isolate that polls `Connectivity().checkConnectivity()` in a `while(true)` loop just to detect a single "went offline" event — while `ConnectivityService` elsewhere in the app already exposes the same thing as a proper stream via `Provider`. This is leftover exploration code from before we found the simpler approach, and it should have been deleted rather than left running alongside its replacement.
- **Naming and language are inconsistent throughout** (`ingresar`/`registrar`/`contrasena` next to `getProducts`/`addToCart`, `productReposirory.dart` misspelled, `ProductDetail_view.dart` breaking Dart file-naming convention). Harmless individually, but it adds friction every time you jump between files, and it's a sign we didn't agree on conventions before splitting up work.
- **The Android package is still `com.example.unimarket`.** Never rebranded from the `flutter create` default — a five-minute fix we just never got back to before the deadline.
- **No CI.** Nothing runs `flutter analyze` or `flutter test` on push; several `analyzer` warnings (unused imports, deprecated `MaterialStatePropertyAll`, etc.) have been sitting in the codebase the whole time because nothing surfaces them automatically.

None of this is a complaint about the course — the constraints (small team, one semester, a rubric with sensor/offline/database requirements to hit) are exactly why the shortcuts exist. But if I were starting UniMarket over as a real product, the data layer and state management are where I'd spend the first week, not the UI.

## Team

Built by [SantiagoBastoss](https://github.com/SantiagoBastoss) and [cracsi](https://github.com/cracsi) for ISIS3510 (Mobile Application Development), Universidad de los Andes, Spring 2024.
