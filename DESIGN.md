# Jourbal — design brief

Personal journaling app. Mac-first (where longform is written), iPhone for check-ins and reading. Client-side encryption with CloudKit as a blob store, optional Claude reflection. Single user, no server of our own.

Mockups: see `design/jourbal-canvas.pdf` (exported from the design canvas). Screen names below match the artboards.

## Stack

- **Apps:** SwiftUI, one Xcode project, iOS + macOS targets. Shared: models, store, sync, crypto, reflection client. Platform-specific: views.
- **Editor (Mac):** `NSTextView`-backed; markdown source, plain text on disk. Native spellcheck, shortcuts, IME.
- **Local store:** SQLite (GRDB or SwiftData — decide at setup). Source of truth on each device.
- **Sync:** CloudKit private database, used as a dumb store for client-encrypted blobs: one record type `Blob(id, kind, version, updatedAt, device, ciphertext)`. CloudKit never sees plaintext. Behind a `SyncTransport` port so a self-hosted store can be added later without touching anything else.
- **Crypto:** libsodium / CryptoKit. One symmetric data key, stored in Keychain with iCloud Keychain sync so devices share it. Per-record encryption. Never hand-roll primitives.
- **Auth:** Face ID (iOS) / Touch ID (Mac) via LocalAuthentication, gating key access. Mac re-locks after idle (default 5 min).
- **Health:** HealthKit on iOS only (no Health store on macOS). iPhone mirrors State of Mind samples into the local store and writes marks/moods back; Mac sees them via sync.
- **AI:** Claude API called directly from the client; API key in Keychain. Model selectable in settings.

## Data model (plaintext inside encrypted blobs)

```
entry       (id, created_at, updated_at, body_md, device, kind: 'write'|'checkin')
tag         (id, name)                      -- normalised lowercase, rename keeps id
entry_tag   (entry_id, tag_id)
mood        (id, at, kind: 'daily'|'momentary', valence -1..1, labels[], associations[], source: 'app'|'health', health_uuid?)
mark_kind   (id, name, health_workout_type?)
mark        (id, kind_id, at, note?, entry_id?)
reflection  (id, created_at, prompt, model, scope_description, transcript_blob?)
pin         (id, reflection_id, entry_id?, range_start?, range_end?, excerpt, note?)
schedule    (id, kind: 'checkin'|'write', times[], days[], until?, deliver_to, skip_rules{})
```

Rules:
- AI material never lives in `entry.body_md`. Pins are attached, attributed, immutable (notes on a pin are editable; the excerpt is not).
- Mood: Health is the interoperable log; the app keeps a mirrored copy for display/query. Writes go both ways (app → Health, Health → app).
- Tags: added from a tag row under the entry header (type-ahead over existing tags) or inline as `#tag` in the body (completion, same as `/mark`). Inline tokens are parsed into `entry_tag` on save; the literal text stays in the body. Tags filter the timeline, are a reflection scope ("tagged work"), and are a search facet. No tag hierarchy, no colours.
- Marks have no streaks/targets. Created from check-in chips, `/name` in the editor (also inserts a line), or Year view.
- Export "my words only" must be possible: entries + marks, no reflections/pins.

## Screens

**Mac**
- *Writing:* sidebar timeline (month groups, mood dot per day, amber dot = has pin), editor centered 620px, header has date/time + daily mood (5-step) + "feeling now" emotion chips + tag row (`#office-move`, type-ahead, `+ tag`), status bar (words, sync state). Opens to today's entry with cursor at end after Touch ID (creates one if none).
- *Reflection pane* (sidebar collapses): scope chips (this entry / this week / month / same week last year / tag), word count, names-replaced count, "include moods" off by default, prompt chips (user-editable), chat exchange, every reply has **Pin to this entry** and **Write about this**. Footer: "Not kept unless you pin" · Keep whole conversation · Discard.
- *Pinned excerpt:* amber card below entry body, sans-serif (entry body is serif), header "PINNED · CLAUDE · time · scope", note field, Open conversation. Timeline filter: All / Only my words / Pinned.
- *Settings → Schedule:* check-in (once/twice daily, times with ±20 min jitter, until-date then ask, deliver to iPhone/Mac/active, on tap → straight into check-in, skip rules), writing reminder (days, time, Mac only, skip if 100+ words today), After Touch ID (open to today / where I left off / timeline; re-lock timer).
- *Year:* week×day grid coloured by daily mood (toggle: emotion / words written), white dot = has mark, per-mark strips below on the same axis, sidebar with emotion/association/mark counts, Print poster.

**iPhone** (dark mode shown; both themes follow system)
- *Check-in:* opened from notification, already unlocked. Daily mood (5), "Right now" emotion chips, "Mostly about" association chips, optional 1–2 lines, Log to Apple Health (on by default), Save. Target: ~20 s.
- *Timeline:* grouped by week, mood dot, serif preview, filter All / Only my words / Pinned. Tabs: Timeline · Check in · Search.
- *Entry:* read + short edit, pinned card, check-ins appended under the day.

## Visual

- Two typefaces: **serif (Newsreader) = your words**, **sans (Inter) = app and AI**. Never mix.
- Light: bg #FFFFFF, panel #F6F6F8, text #1C1C1E, muted #6B6B70, accent #3B5BDB, AI amber #B45309 on #FBF1E6.
- Dark: bg #151517, panel #1C1C1E, text #ECECEE, muted #9A9AA0, accent #9DB0FF, AI amber #F0A24A on #241B12.
- Mood scale (differs in lightness, not just hue): Low #5B4A8A · Flat #8C8C94 · Mixed #C9A227 · Steady #5B9A5B · Good #3B5BDB. Dark variants: #7E6BB5 · #9A9AA0 · #D9B648 · #7FBF7F · #9DB0FF.

## Non-goals (v1)

- Sharing, multi-user, web client.
- Our own server. CloudKit holds ciphertext only; a self-hosted transport is a later adapter, not v1.
- Relying on CloudKit's encryption instead of our own. Client-side encryption is not negotiable.
- Local LLM (later, for summaries).
- Streaks, goals, habit scoring.

## Build order

1. Store + crypto + CloudKit blob transport. Round-trip encrypted entries between two devices; verify in CloudKit Dashboard that records are ciphertext only.
2. Mac: editor, timeline, Touch ID, open-to-cursor.
3. iPhone: check-in + notifications + schedule; HealthKit mirror (read + write State of Mind).
4. Marks (chips, `/name`, Year strips).
5. Year view.
6. Reflection pane: scope → send → pin. Name aliasing before anything leaves the device.
