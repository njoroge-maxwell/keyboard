# LightKeys – ultra-light iOS-style keyboard for Android 11 (Kotlin, no Compose, offline)

Open the folder in Android Studio (Koala+ / JDK 17) and let it sync, or run `gradle wrapper && ./gradlew assembleRelease`.
`minSdk 26`, `compileSdk 35`, `targetSdk 30` (raise to 34+ only if you publish to Play).
Release APK: R8 + resource shrinking, one English resource set, a single AndroidX dependency
(`activity-ktx`, used only for the document picker). No native code, so ABI splits are irrelevant. No `INTERNET` permission.

## Enable on Android 11
1. Install the APK, open **LightKeys**.
2. Tap **1. Open input method settings** → switch LightKeys on (accept the warning).
   Manual path: Settings → System → Languages & input → On-screen keyboard → Manage on-screen keyboards.
   On ColorOS/OPPO it is Settings → Additional settings → Keyboard & input method.
3. Tap **2. Choose LightKeys** and pick it in the system dialog. Any text field also offers the picker through the keyboard icon in the nav bar.

## Bringing your Gboard words over
Gboard's learned-word database sits in Gboard's private app storage. Android 11 sandboxing means no other app can read it without root, so there is no silent import. The two supported bridges:

1. **Gboard export file** – in Gboard: Settings → Dictionary → Personal dictionary → (language) → ⋮ → **Export**. Then in LightKeys: **Import Gboard export** and choose the `.zip` (or the `dictionary.txt` inside it).
   The importer streams the file (`# Gboard Dictionary version:1`, then `shortcut<TAB>word<TAB>locale`) into a SQLite table whose primary key is the lookup index. Each entry is indexed under its shortcut and its word. Lookup = one range scan with `LIMIT 3`, on a background thread.
   Note: Gboard exports your *personal dictionary* (words and shortcuts you added). Words it silently auto-learned are not in the export.
2. **Android personal dictionary** – tap **Allow Android personal dictionary** (runtime `READ_USER_DICTIONARY`). Words and shortcuts there are queried through `UserDictionary.Words` and merged into the same 3-slot strip.

## Memory design
- `KeyboardView` is one custom view; all `Paint`/`RectF`/`Path`/`FontMetrics` objects are pre-allocated, nothing is allocated in `onDraw()` or the touch handlers (up to 4 simultaneous fingers).
- Frosted look = semi-transparent solid fills + 1dp borders + a drawn bottom lip. No blur, no shadow bitmaps, no extra windows (key popups are drawn inside the view).
- Emoji drawer: nothing exists until opened; it draws only visible rows, parses one category at a time, and `release()`s on close, `onWindowHidden()` and `onTrimMemory()`. The SQLite handle is closed on hide/trim too.
- Suggestions are disabled in password, e-mail and URL fields and fields that set `NO_SUGGESTIONS`.

## Emoji rendering – read this
The drawer uses the **system emoji font** by default (Noto on Android 11; the list is kept to Emoji 12.0 so nothing renders as tofu). Apple's emoji artwork is proprietary and is not bundled.
To use image glyphs inside the drawer, add 64×64 PNGs named by lowercase hex code points joined with `-` (`1f600.png`, `1f44d-1f3fb.png`) to `app/src/main/assets/emoji/` (or `files/emoji/` at runtime). They are decoded lazily into a 3 MB `LruCache` and evicted when the drawer closes. Use an open set (OpenMoji, Noto) or art you hold rights to.
Apps and text fields still render committed emoji with the system font, unless the device has a system-level emoji font module.

## Known limits / honest notes
- I could not run Gradle or an Android SDK in this environment, so the project has not been compiled or tested on a device. Expect to fix a few typos or warnings on first sync.
- "iOS 27" has no public design spec to copy; the look is an approximation of the current iOS keyboard (key proportions, lip, popup, adaptive return key, colours). Tune the numbers in `KeyboardView.setDims()` / `applyThemeColors()`.
- No autocorrect model, no swipe typing, no long-press accents, and no text-shortcut expansion on space (tap the suggestion instead). The strip is dictionary-prefix only, by design, to stay inside the size budget.
