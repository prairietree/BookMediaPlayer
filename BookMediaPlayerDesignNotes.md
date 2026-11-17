# BookMediaPlayer — Design Notes

## App Description

An Android app for playing long-form audio (audiobooks, lectures, podcasts) in
MP3 and other common audio formats. The core feature is per-book playback
position tracking — the app remembers exactly where you stopped in every book
and resumes from that point automatically.

**Main screen (Library):** a simple list of every book the user has opened,
each row showing the title and a bit of progress info (e.g. elapsed/total time
or percent complete, last-played date). The currently active (or most recently
played) book is always pinned at the top of the list, with the remaining books
ordered by how recently they were played (`lastPlayedAt` descending).

**Opening a new book:** the user browses and picks one of:
- a single audio file (one-shot book)
- a folder containing audio files (treated as one book, tracks played in
  natural filename order)
- a playlist file (e.g. `.m3u`) inside a picked folder (picking the parent
  folder via SAF is required so the app has read access to the referenced
  audio files)

Whichever is picked gets added to the library as a new book entry and becomes
the active book. The same file/folder/playlist can be opened again later as a
separate, independent book entry (e.g. to track a second listen-through) —
each entry has its own resume position. If another book is currently playing
when the user opens a different book, the current book's position is saved and
playback pauses before switching to and auto-resuming the newly selected book.

Opening an existing library entry silently resumes playback from its saved
position — no confirmation prompt. To re-orient the listener after a break,
resuming automatically backs up a little from the saved position based on how
long it's been since `lastPlayedAt`:
- more than 3 hours since last played → back up 5s
- more than 3 days since last played → back up 15s (instead of the 5s above)
- otherwise → resume at the exact saved position

The player screen also offers explicit "restart" actions to jump back to the
start of the current track (for multi-file/playlist books) or the start of
the current chapter (if chapter metadata exists), without disturbing the
auto-tracked resume position until the user plays on from there.

### Feature ideas to consider for the above flows

- Progress display format — elapsed/remaining time vs. percent vs. both.
- Library sorting/filtering — recently played, alphabetical, in-progress vs. finished.
- Mark a book finished / remove from library (without deleting the underlying file).
- Search within the library.
- Skip forward/back buttons with a user-configurable interval (e.g. 15s/30s).
- What happens if a file is added to/removed from a folder after it's been added as a book (re-scan behavior).
- Handling moved/deleted/renamed source files gracefully (flag in library instead of crashing).
- Light/dark theme.
- Home screen widget for quick resume of the most recent book.
- Listening stats (optional) — total time listened, books finished.

## Target Platform

- **Language:** Kotlin
- **UI:** Jetpack Compose
- **Min SDK:** 34 (Android 14) — only actively supported Android versions need support
- **Target/Compile SDK:** latest stable (35/36)
- **Build system:** Gradle (Kotlin DSL, `build.gradle.kts`)

## Architecture

MVVM, single `:app` module:

```
Compose UI  -->  ViewModel  -->  Repository  -->  Room (book metadata/position)
                                            \-->  MediaController -> MediaSessionService (Media3/ExoPlayer)
```

- **UI layer:** Compose screens (Library, Player, Settings) + ViewModels exposing `StateFlow`.
- **Playback layer:** `androidx.media3` (`media3-exoplayer`, `media3-session`, `media3-common`).
  A `MediaSessionService` runs playback in a foreground service (`foregroundServiceType="mediaPlayback"`),
  with audio focus & ducking enabled (`setAudioAttributes(..., handleAudioFocus = true)`), which gives us:
  - background playback
  - lock screen / notification controls
  - Bluetooth & headset button handling
  - automatic pause/duck and position save on calls or navigation prompts
  - Android Auto compatibility (future)
- **Data layer:** Room database for book library + playback state; Jetpack DataStore
  (Preferences) for app-wide settings (default playback speed, sleep timer default, theme).
- **DI:** Manual service locator (kept intentionally simple — no Hilt/Dagger unless it
  becomes a pain point).

## Data Model (draft)

`Book` entity (Room):
| Field | Type | Notes |
|---|---|---|
| `id` | Long (PK, autogenerate) | internal book id — the same source can be added more than once as separate book entries |
| `sourceUri` | String | persisted content URI (SAF) of the file or folder — not unique |
| `sourceType` | Enum (`FILE`, `FOLDER`, `PLAYLIST`) | how to open and refresh `sourceUri` |
| `title` | String | from tags or filename |
| `author` | String? | from tags if available |
| `durationMs` | Long | total length (sum of tracks for multi-file books) |
| `currentTrackIndex` | Int | 0-based index into `BookTrack` (always 0 for single-file books) |
| `trackPositionMs` | Long | resume position within `currentTrackIndex` |
| `lastPlayedAt` | Long? (epoch) | nullable for never-played books; drives Library ordering |
| `createdAt` | Long (epoch) | fallback ordering for never-played books |
| `playbackSpeedOverride` | Float? | per-book speed, falls back to global default |
| `isFinished` | Boolean | marks completed books for filtering and resume behavior |
| `isUnavailable` | Boolean | flags missing/moved files or revoked SAF permissions |

`BookTrack` entity (Room, 1-to-many with `Book`):
| Field | Type | Notes |
|---|---|---|
| `id` | Long (PK, autogenerate) | internal track id |
| `bookId` | Long (FK -> `Book.id`) | owning book (cascade delete) |
| `trackIndex` | Int | 0-based playback order (from natural filename sort or playlist) |
| `uri` | String | resolved content URI for this track |
| `title` | String | from tags or filename |
| `durationMs` | Long | cached track duration so total progress/seeking doesn't re-scan SAF |

## Playback Position Tracking

- Single `(currentTrackIndex, trackPositionMs)` resume point per book (no multi-bookmark support for now).
- Save `(currentTrackIndex, trackPositionMs)` (and update `lastPlayedAt`) to Room:
  - on pause / stop (only if playback actually occurred since the last explicit restart jump, so a paused "restart track/chapter" doesn't overwrite the saved position)
  - periodically (~10–15s) while playing, in case the process is killed
  - in the service's `onTaskRemoved` / lifecycle teardown
- On open, silently seek and resume — no confirmation prompt. If `lastPlayedAt != null` and `!isFinished`,
  the seek target is backed up from `trackPositionMs` based on elapsed time since `lastPlayedAt`:
  - `now - lastPlayedAt > 3 days` → back up 15s
  - `now - lastPlayedAt > 3 hours` → back up 5s
  - otherwise → resume at `trackPositionMs` exactly
  (clamped to `0` of the current track—or optionally stepping back into the previous track if near a boundary—never negative).
- Explicit restart actions (separate from the tracked resume position) let the user jump
  back to the start of the current track (relevant for multi-file/playlist books), or the
  start of the current chapter if chapter metadata is present.
- Chapter markers: Media3 can extract ID3 chapter metadata automatically where present.
- Folder/playlist track order: natural filename sort (numeric-aware, e.g. `Track 2` before `Track 10`; no track-number tag parsing for now).

## File Access & Permissions

- Storage Access Framework (`ACTION_OPEN_DOCUMENT_TREE` / `ACTION_OPEN_DOCUMENT`) to
  let the user pick audiobook files/folders.
- `takePersistableUriPermission` so picked files remain accessible across app restarts
  without re-prompting.
- No broad storage permission needed given `minSdk 34+` and scoped storage.
- Android 14+ foreground service & notifications: declare `FOREGROUND_SERVICE` and
  `FOREGROUND_SERVICE_MEDIA_PLAYBACK` in the manifest, and request `POST_NOTIFICATIONS`
  at runtime so lock-screen/notification media controls appear reliably.

## Key Libraries

| Purpose | Library |
|---|---|
| Playback engine | `androidx.media3:media3-exoplayer` |
| Background service / notification / media session | `androidx.media3:media3-session` |
| Shared media types | `androidx.media3:media3-common` |
| Local database | `androidx.room` |
| Settings storage | `androidx.datastore:datastore-preferences` |
| Async | Kotlin Coroutines & Flow |
| UI | Jetpack Compose |

## Open Questions / Sections To Flesh Out Later

- [ ] **UI/UX flows** — screen list, navigation graph, wireframes for Library/Player/Settings.
      - *Draft answer:* Three top-level screens/destinations: **Library** (start destination),
        **Player**, **Settings**, navigated with Compose Navigation (single-activity). Library
        always pins the active (or most recently played) book at the top of the list and sorts
        the rest by `lastPlayedAt` descending; tapping a row opens Player for that book
        (saving/pausing any other active book first, then auto-resuming per the rules above).
        A FAB/menu on Library opens the "add book" flow (file or folder picker). Player → back
        button returns to Library; a toolbar icon opens Settings. No deep wireframes yet —
        just this screen graph for now.
- [ ] **Import/scanning flow** — how books are added (single file vs. folder vs. multi-file
      "chapters" treated as one book), de-duplication, re-scan behavior.
      - *Draft answer:* Adding is always an explicit user action via SAF picker (file or
        folder; `.m3u` playlists require picking their parent folder via
        `ACTION_OPEN_DOCUMENT_TREE` so SAF grants read access to the referenced tracks) — no
        background/automatic library scanning. Folder picks are scanned non-recursively for
        supported audio extensions, sorted with natural filename sort, and cached in
        `BookTrack` at add time only. No automatic de-duplication — re-adding the same URI
        intentionally creates a second, independent `Book` row (per the "separate
        listen-through" requirement already noted). Re-scan of a folder's contents only
        happens on explicit user request ("Refresh" action on that book), not automatically.
- [ ] **Audio format support matrix** — confirm which formats/codecs to officially support
      (MP3, M4A/AAC, OGG, FLAC, WAV) and any container edge cases.
      - *Draft answer:* Support whatever Media3/ExoPlayer's default extractors+decoders handle
        out of the box on-device: MP3, M4A/AAC, OGG/Opus/Vorbis, FLAC, WAV. No custom/FFmpeg
        extension initially. Unsupported/corrupt files are skipped during folder scan with a
        visible warning rather than blocking the whole import.
- [ ] **Playback features** — sleep timer, variable speed range, skip-silence, rewind-on-resume
      (e.g. auto-rewind 5–10s when resuming after a long pause).
      - *Draft answer:* v1 includes a simple sleep timer (preset durations: 15/30/45/60 min +
        "end of chapter/track"), variable speed 0.5x–3.0x in 0.1x steps (global default +
        per-book override, matching the `playbackSpeedOverride` field), and the auto-rewind on
        resume already specified in the Playback Position Tracking section. Skip-silence is a
        nice-to-have, deferred post-v1.
- [ ] **Multi-file books** — books split across many mp3 files (common for ripped CDs):
      how position tracking spans file boundaries.
      - *Draft answer:* Tracks are loaded into Media3 via `Player.setMediaItems(...)` using
        the cached `BookTrack` rows. Resume state is stored as `(currentTrackIndex,
        trackPositionMs)` rather than a single global offset (avoiding VBR duration drift and
        matching Media3's per-item `currentPosition`), while total elapsed/remaining time in
        the UI is computed from the cached `BookTrack.durationMs` values.
- [ ] **Backup/restore** — exporting library + positions (local file export, or cloud sync later).
      - *Draft answer:* v1 ships a manual "export/import library" action that serializes the
        Room `Book`/`BookTrack` tables (+ settings) to a single JSON file via SAF, for local
        backup or moving to a new device. Because `content://` SAF URIs are device- and
        permission-specific, cross-device restore pairs with the "re-locate" flow to re-link
        books to their files on the new device. No automatic/cloud sync in v1 — revisit if
        there's demand.
- [ ] **Error handling** — missing/moved files, revoked SAF permissions, corrupt metadata.
      - *Draft answer:* On open/playback failure (file missing, permission revoked, unreadable
        metadata), mark `Book.isUnavailable = true` instead of crashing, surface a
        "file unavailable — re-locate?" state on its Library row, and let the user re-point it
        to a new URI (preserving `currentTrackIndex` / `trackPositionMs`) or remove the entry.
- [ ] **Testing strategy** — unit tests for position-tracking logic, instrumentation tests for
      the playback service.
      - *Draft answer:* Unit tests (JVM, no device) for the resume-offset calculation, Room
        DAOs, and ViewModel state; instrumentation tests (`androidx.media3.test.utils` +
        Compose UI testing) for the `MediaSessionService` lifecycle and basic Player-screen
        interactions. CI runs unit tests on every push; instrumentation tests run on
        PR/pre-release only given emulator cost.
- [ ] **App icon / branding / naming** — final app name, icon, theming (light/dark).
      - *Draft answer:* Working title "BookMediaPlayer" stands as the app name for now;
        final name/icon/branding deferred until closer to a Play Store listing. Theming
        follows Compose Material3 dynamic color with manual light/dark override in Settings
        (already listed as a feature idea).
- [ ] **Release process** — signing config, versioning scheme, Play Store listing requirements.
      - *Draft answer:* Standard Gradle Play App Signing (upload key + Play managed signing),
        semantic versioning (`versionName` = `major.minor.patch`, `versionCode` auto-incremented
        per release). Internal testing track first, then closed/open testing before production.
        Play Store listing assets (screenshots, description, privacy policy re: SAF file access)
        to be drafted once UI is stable.
- [ ] **Accessibility** — TalkBack support for player controls, content descriptions.
      - *Draft answer:* All interactive controls (play/pause, skip, speed, sleep timer,
        library rows) get explicit `contentDescription`/semantics in Compose; player screen
        state (position, title) exposed via `liveRegion`/semantics so TalkBack announces
        changes. Validate with TalkBack manually each release rather than automated a11y tests
        initially.
- [ ] **Performance** — cold start time, large library scanning performance.
      - *Draft answer:* Target sub-second cold start to a visibly interactive Library screen
        (Room query on a background dispatcher, no blocking I/O on the main thread). For
        folder imports, query `ContentResolver` directly via
        `DocumentsContract.buildChildDocumentsUriUsingTree` (avoiding slow
        `DocumentFile.fromTreeUri()` wrappers) off the main thread with a progress indicator,
        and cache the track list/durations in `BookTrack` so subsequent opens are instant.
