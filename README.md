# QUICK Zi STORE (Flutter, Android)

Store-owner app. This step contains the **Login / Register** flow, location
permission gate, glass UI, and the animated isometric wallpaper.
Package name: `Store.QUICKZi.store` (matches `android/app/google-services.json`).

## Features in this build

| Screen | What it does |
|---|---|
| Startup | Splash, Remote Config fetch, location permission gate, auto-login check |
| Login gate | Glass buttons: **Login** / **Register**, animated isometric wallpaper |
| Register | Store name, owner name, phone (+91), store location (map pin, drag, **Done**), **Enter Access Key**, **Request key** |
| Login | Phone (+91), **Request key**, enter 14-char key, **Login**. Unregistered number shows **Register First** |
| Home | Restored profile (owner, store, phone, location map). Products / Orders / Payments / Riders are placeholders for the next step |

### Access key rules
- 14 characters, letters + digits (ambiguous `0 1 I L O` removed).
- Sent to the Telegram chat from Remote Config via Bot API `sendMessage`.
- Valid for **5 minutes**. Resend unlocks after **30 seconds**.
- Max 5 wrong attempts per key, then a new key must be requested.
- The key is kept only in app memory. It is never written to Firestore.

### Session
- Profile is saved in Firestore at `stores/{phone}`.
- Phone number is kept in local storage, so the user stays logged in until
  **Logout** or until app data is cleared.

## 1. Firebase setup (one time)

1. **Firestore Database**: create it (Native mode). For now, use the rules in
   `firestore.rules` (read the warning inside the file).
2. **Remote Config**: add two parameters and click **Publish changes**:
   - `telegram_bot_token` : your bot token from BotFather (`123456:ABC...`)
   - `telegram_chat_id`   : numeric chat id (e.g. `-1001234567890` for a group,
     or your own user id)
3. **Telegram bot**:
   - Talk to `@BotFather` → `/newbot` → copy the token.
   - Open a chat with the bot (press **Start**) or add it to a group.
   - Send any message, then open
     `https://api.telegram.org/bot<TOKEN>/getUpdates` to read `chat.id`.

## 2. Build the APK

Requirements: Flutter 3.47.x (tested with 3.47.7), JDK 17+ (tested with 21),
Android SDK 36.

```bash
flutter pub get
flutter analyze
flutter test
flutter build apk --release
```

Output: `build/app/outputs/flutter-apk/app-release.apk`
(the build directory is excluded from the workspace snapshot).

For Play Store signing, create a keystore and add a `signingConfigs.release`
block in `android/app/build.gradle.kts` (currently release uses debug keys so
the APK installs directly).

## Important notes and limitations

- **Telegram token in the client.** Remote Config is readable by the app, so
  the bot token can be extracted from the APK. For production, move the
  Telegram call to a Cloud Function (send the key server-side and return only
  "sent").
- **Key is sent to one fixed chat** (`telegram_chat_id`). If all store owners
  must receive their own keys, each owner needs a separate chat id, or the
  design should change to a per-phone chat mapping.
- **Firestore is open** because there is no Firebase Auth yet. Add Auth before
  launch (see the warning in `firestore.rules`).
- **Map**: uses OpenStreetMap tiles and Nominatim reverse geocoding (no API key).
  Please follow the OSM usage policy for production traffic, or switch to
  Google Maps.
- The 3D wallpaper is a **native Flutter port** of the Three.js "Logic Core"
  scene (isometric platform, glowing core, 12 orbiting nodes). It runs offline
  and does not need a WebView.

## Project layout

```
lib/
  main.dart                      Firebase init, app root
  theme/app_theme.dart           colours, dark theme
  models/store_profile.dart      Firestore model (stores/{phone})
  services/
    access_key.dart              14-char key generator and validation
    auth_service.dart            request key, verify, register, login, session
    config_service.dart          Remote Config (telegram_bot_token, telegram_chat_id)
    telegram_service.dart        Telegram Bot API sendMessage
    store_repository.dart        Firestore reads and transactional create
    location_service.dart        permission + GPS
    session_store.dart           local phone session
    app_exceptions.dart          user-facing errors
  utils/                         phone normalisation, input formatters
  widgets/                       glass components, isometric background
  screens/                       startup, login gate, login, register, map picker, home
test/                            unit + widget tests
firestore.rules                  Firestore security rules (see warning)
```
