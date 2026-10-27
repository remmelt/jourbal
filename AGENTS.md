# AGENTS.md — Jourbal

Instructions for any coding agent working in this repository. Read `DESIGN.md` first; it is the product brief and the source of truth for behaviour, data model and visual rules. This file is about *how* the code is built.

## 1. Repository layout

```
.
├── AGENTS.md
├── DESIGN.md
├── design/                      # exported mockups (PDF/PNG); read-only reference
├── Jourbal.xcworkspace
├── Apps/
│   ├── JourbalMac/              # macOS target: AppKit-hosted SwiftUI, menus, windows
│   └── JourbalIOS/              # iOS target: SwiftUI, notifications, HealthKit
├── Packages/
│   ├── JourbalDomain/           # entities, value objects, domain rules. No imports but Foundation.
│   ├── JourbalApplication/      # use cases + ports (protocols). Imports Domain only.
│   ├── JourbalAdapters/         # one library target per adapter (see §3)
│   └── JourbalTestKit/          # fakes, builders, fixtures for tests
├── scripts/                     # build, test, lint, format wrappers
└── docs/adr/                    # architecture decision records, 0001-...md
```

Swift code lives in SPM packages; the app targets are thin. If a type does not obviously belong to an app target, it belongs in a package.

## 2. Hexagonal architecture (Swift)

Dependency direction, strictly: `Apps → Adapters → Application → Domain`. Nothing points inward-to-outward. Enforced by package dependencies, not by discipline.

### Domain (`JourbalDomain`)
- Pure Swift. `import Foundation` only. No SwiftUI, no SQLite, no HealthKit, no Combine.
- Entities as `struct`s with value semantics: `Entry`, `Tag`, `Mood`, `Mark`, `MarkKind`, `Reflection`, `Pin`, `Schedule`.
- `Tag` is a normalised name (`lowercased`, trimmed, `[a-z0-9-]`, 1–40 chars) with a stable `TagID`; renaming a tag keeps the id. `Entry.tags: Set<TagID>`. Inline `#tag` tokens in the body are a *presentation* affordance that the editor resolves to `Entry.tags`; the body keeps the literal text so an export without the database still carries them.
- Identifiers are typed: `struct EntryID: Hashable, Codable { let raw: UUID }`. Never pass bare `UUID`s or `String`s as ids.
- Domain rules live here as methods or free functions: e.g. `Schedule.nextFiring(after:calendar:)`, `Pin.validate()` (excerpt non-empty, scope description present), `Mood.valence` clamped to `-1...1`.
- Invariants are enforced in initialisers; an invalid entity cannot be constructed.
- Errors: one `enum DomainError: Error` per package, exhaustive, no `String` payloads for control flow.

### Application (`JourbalApplication`)
- **Ports** are protocols, named by role, not technology:
  - Driven (outbound): `EntryRepository`, `TagRepository`, `MoodRepository`, `MarkRepository`, `ReflectionRepository`, `ScheduleRepository`, `SyncTransport`, `Cipher`, `KeyStore`, `Authenticator` (biometrics), `HealthStore`, `Notifier`, `ReflectionProvider` (the LLM), `Clock`, `Calendar provider`.
  - Driving (inbound): use cases, one type each, e.g. `CreateEntry`, `AppendCheckIn`, `TagEntry`, `RenameTag`, `RecordMood`, `AddMark`, `AskReflection`, `PinExcerpt`, `SyncOnce`, `MirrorHealthMoods`, `ScheduleCheckIns`.
- A use case is a `struct` with injected ports and a single `callAsFunction` or `execute`. It returns a value or throws. No side channels.
- Use cases are `Sendable` and run on whatever actor the caller chooses; repositories are actors or `Sendable` classes.
- No port exposes an adapter type. `EntryRepository` returns `Entry`, never a GRDB `Row` or a `NSManagedObject`.
- DTOs between Application and Adapters only when the domain shape is not suitable for transport (sync envelope, Health sample mapping). Keep them in the adapter.

### Adapters (`JourbalAdapters/*`)
One target per adapter, depending on `JourbalApplication` and the third-party framework only:

| Target | Implements | Notes |
|---|---|---|
| `SQLiteStore` | all `*Repository` | GRDB. Migrations in `Migrations.swift`, numbered, never edited after merge. |
| `SodiumCipher` | `Cipher` | libsodium secretbox (XChaCha20-Poly1305). Per-record nonce. Nothing else. |
| `KeychainKeyStore` | `KeyStore` | `kSecAttrSynchronizable = true`, `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` is **wrong** here (breaks sync); use `WhenUnlocked`. Access guarded by biometrics via `LAContext`. |
| `BiometricAuthenticator` | `Authenticator` | LocalAuthentication. |
| `CloudKitSyncTransport` | `SyncTransport` | Private database, one `Blob` record type, ciphertext in a `CKAsset` (blobs can exceed the 1 MB field limit). Change tokens for incremental fetch; `CKModifyRecordsOperation` with `.ifServerRecordUnchanged` and version-based conflict resolution in the use case, not here. |
| `HealthKitStore` | `HealthStore` | iOS only. `#if canImport(HealthKit)`. State of Mind read/write, workout write for marks. |
| `UserNotificationsNotifier` | `Notifier` | UNUserNotificationCenter; schedules local notifications from `Schedule`. |
| `ClaudeReflectionProvider` | `ReflectionProvider` | Claude API via URLSession. Applies name aliasing *before* building the request. |
| `SystemClock` | `Clock` | and `FixedClock` in TestKit. |

Rules:
- Adapters are replaceable. If replacing one requires touching Application or Domain, the port is wrong; fix the port.
- Adapters do not call each other. Composition happens in the app target's `CompositionRoot`.
- Every adapter has a contract test in `JourbalTestKit` that any implementation of the port must pass (`EntryRepositoryContract`), run against both the real adapter and the in-memory fake.

### Apps
- `CompositionRoot.swift` builds the object graph once at launch. No service locator, no singletons, no `@EnvironmentObject` carrying repositories around.
- View models (`@Observable` classes) depend on use cases, never on ports or adapters.
- Views depend on view models. Views contain no business logic and no formatting beyond presentation.

## 3. macOS best practices

The Mac app is the primary app. It should feel native, not like an iPad app in a window.

**Structure**
- `App` with `WindowGroup` for the journal window and `Settings` scene for preferences. One document-less window; reopening the app reuses it.
- Keyboard first. Every action in a menu has a shortcut. `Commands` builder for the menu bar: File (New entry ⌘N, Export…), Edit (standard), View (Toggle Sidebar ⌘⌥S, Toggle Reflection ⌘⇧R, Year ⌘2, Write ⌘1), Go (Today ⌘T, Previous/Next day ⌘[ ⌘]), Window, Help.
- Focus and first responder matter: after unlock, the editor is first responder with the insertion point at the end of today's entry. Test this; it is a stated requirement.
- Sidebar uses `NavigationSplitView` with `.navigationSplitViewStyle(.balanced)`; toolbar via `.toolbar` with `ToolbarItem(placement: .navigation / .primaryAction)`.

**Editor**
- Wrap `NSTextView` in `NSViewRepresentable`. Do not use SwiftUI `TextEditor` for the body; it lacks spellcheck integration, proper undo, and native shortcuts.
- Keep the text storage as the single source of truth while editing; debounce persistence (500 ms idle, and on focus loss, window close, app termination, sleep).
- Markdown is stored as plain text. Rendering (bold, headings) is done via `NSTextStorage` attributes, never by rewriting the source.
- Respect system text settings: `NSFont.preferredFont(forTextStyle:)` scaled; honour "Reduce Motion", "Increase Contrast".

**Appearance**
- Use semantic colors (`Color(nsColor: .textBackgroundColor)`, `.labelColor`, `.secondaryLabelColor`) for chrome; only the mood scale and AI amber are literal, and both are defined once in `Theme.swift` with light/dark variants from `DESIGN.md`.
- Never read `colorScheme` to pick colors manually when a semantic color exists. Dark mode must work with zero code paths that branch on it, except the two literal palettes above.
- Typography: entry body uses the serif (Newsreader, bundled, licence in `Resources/Fonts/`); everything else uses the system font. Do not introduce a third face.

**Security & privacy**
- App Sandbox on. Entitlements: `keychain-access-groups`, `com.apple.security.network.client` (Claude API), iCloud with CloudKit, and nothing else for the Mac. HealthKit entitlement on iOS only.
- Hardened Runtime on; notarised for distribution even though it is a personal app (Gatekeeper, and it forces signing hygiene).
- Lock state is a domain concept (`LockState`), not a view flag. Re-lock on idle timer, screen lock (`NSWorkspace.sessionDidResignActiveNotification`), and sleep. On lock: drop the data key from memory, blur the window content, require Touch ID to resume.
- No analytics, no crash reporters that upload. Logging via `os.Logger` with privacy annotations (`\(entry.id, privacy: .public)`, bodies never logged).
- Plaintext never touches disk outside the SQLite file, and the SQLite file is encrypted at the record level (ciphertext columns) — SQLite's own file is not the trust boundary.

**Platform hygiene**
- Support the current macOS and one back. No Catalyst.
- Handle window restoration, multiple displays, full screen, and Stage Manager without special-casing.
- Accessibility: every control has a label; VoiceOver can read the timeline and the pinned cards; the mood picker is a radio group with names, not colored dots alone.
- Respect `NSApp.effectiveAppearance` changes mid-session (system auto dark mode switches at sunset).

### 3b. The bar: what a senior Mac engineer checks first

These are the things that get an app rejected in review by someone who has shipped Mac software. Treat each as a requirement, not a nicety.

**Responder chain and menus**
- Actions are `@objc` selectors or SwiftUI `Commands` that resolve through the responder chain, so the same menu item works whether focus is in the editor, the sidebar or the reflection pane, and is disabled (not hidden) when it does not apply. `validateMenuItem` / `.disabled(!canPerform)` must be correct for every item.
- Cmd-Z works across everything: typing, mood changes, tag add/remove, pin notes. Use the window's `NSUndoManager`; register undo for model mutations in the view model, grouped per user gesture. Undo of text uses `NSTextView`'s own manager, bridged, not a second one.
- Cut/copy/paste/select-all go through the standard editing selectors. Paste of rich text is stripped to plain text + markdown.
- Escape closes the reflection pane; Cmd-. cancels an in-flight Ask.

**Text editing**
- TextKit 2 (`NSTextLayoutManager`). Lay out the editor with `NSTextView.scrollableTextView()`; set `textContainerInset` for the 620 pt column rather than wrapping in extra views.
- Typing attributes are stable: setting the serif face and size once in `typingAttributes`, not re-applied on every change. Markdown emphasis is applied by an `NSTextContentStorageDelegate`, so the stored string is never mutated for styling.
- `#tag` and `/mark` completion uses `NSTextView.complete(_:)` with a custom `completions(forPartialWordRange:)`, so it looks and behaves like system completion (Esc to dismiss, arrow keys, Return to accept). No floating custom popover.
- Spellcheck, smart quotes, data detectors: on, and respecting the user's System Settings; smart dashes converted on input, stored as typed.
- Continuous spell checking and automatic text replacement are left enabled; this is prose, not code.
- Insertion point placement after unlock is done via `setSelectedRange` *after* layout completes (`NSTextLayoutManager.ensureLayout`), not on `onAppear`, which fires too early.

**Windows and scenes**
- `NSWindow.restorable` with `restorationClass`; sidebar width, pane visibility and the selected day come back after relaunch. Window position is restored by AppKit; do not store it yourself.
- Toolbar is a real `NSToolbar` (`.toolbarStyle(.unified)`) so customisation, overflow and full-screen behave. Sidebar gets `.sidebar` material and collapses with the standard animation and shortcut.
- Settings is a `Settings` scene with a `TabView` (General, Schedule, Mood & Health, Reflection, Sync, Devices), fixed width, no scrolling tabs.
- Secondary windows (Year, reflection history) are `Window` scenes with ids, opened via `openWindow`, so Cmd-` cycling and the Window menu are correct for free.

**System integration that costs little and reads as native**
- App Intents: `Log mark`, `Log mood`, `New entry`, `Open today`. This gives Shortcuts, Spotlight actions and the menu bar extra without extra UI.
- Menu bar extra (`MenuBarExtra`, optional, off by default) with today's mood and the four most-used marks.
- Services menu: "Add to Jourbal" for selected text.
- Printing: `NSPrintOperation` from a dedicated print view for the Year poster, with page setup, not a PDF rendered from the on-screen view.
- Quick Note / Handoff: entry continues from iPhone to Mac via `NSUserActivity` with the entry id; the Mac opens it with the cursor at the end.
- Notifications on the Mac use `UNUserNotificationCenter` with actions ("Write now", "Later"); clicking lands in the right entry.

**Performance and feel**
- First window appears in under 300 ms on an M-series Mac with the timeline rendering placeholders while decryption pages in. Measure with `os_signpost` and keep the signposts.
- Scrolling the timeline is `List` with stable identities, not a `ScrollView` of `VStack`s; 2,000 entries must not drop frames.
- No layout work on the main actor beyond SwiftUI's own. Decryption, search and sync are off-main.
- Animations use system durations; `reduceMotion` respected.

**Dark mode and vibrancy**
- Sidebar and panes use materials so desktop tint shows through; content area uses `.textBackgroundColor`. Verify with the Accessibility "Increase contrast" and "Reduce transparency" settings on.
- Images and the mood palette have `NSAppearance`-aware asset catalog variants, not runtime branching.

**Distribution hygiene**
- Signed with a Developer ID, hardened runtime, notarised, stapled; `codesign --verify --deep --strict` and `spctl --assess` run in CI.
- Sandbox entitlements are the minimum set and listed in `docs/entitlements.md` with a reason each.
- Crash logs come from the user's Console, nowhere else.

## 4. Sync (no server of our own)

There is no backend. Sync is a dumb blob store behind the `SyncTransport` port; v1's adapter is CloudKit's private database. The design must stay such that a self-hosted transport could be added as a second adapter with no change above the port.

- **Envelope** (what every transport moves): `Blob(id, kind, version, updatedAt, device, ciphertext)`. `kind` is the domain table name or `tombstone`. `version` is a per-record monotonic counter bumped by the writer.
- **Conflict rule** lives in `SyncOnce` (Application), not in the transport: higher `version` wins; equal versions → later `updatedAt`; equal both → lexically greater `device` (deterministic, never asks the user). Entries are append-mostly so this is rarely hit; when it is for an `entry`, keep both bodies and mark the loser as a `conflict` sibling rather than discarding text.
- **Cursor**: the transport returns an opaque cursor (`CKServerChangeToken` for CloudKit); `SyncOnce` persists it per transport.
- **CloudKit specifics** (adapter only): a single custom zone so change tokens work; `CKAsset` for ciphertext; subscription to the zone for push-triggered sync on iOS; `accountStatus` checked before every run and surfaced as a domain `SyncState`, never as an alert from the adapter.
- **Plaintext never reaches CloudKit.** No searchable fields, no titles, no dates beyond `updatedAt` (needed for conflict resolution; acceptable leak, documented in `docs/threat-model.md`).
- **Backup**: a local, user-triggered export of the whole store as encrypted blobs plus a separate "my words only" plain-text export. A scheduled reminder to run it quarterly is a `Schedule` like any other.
- **Verification**: an integration test against a CloudKit development container (`CKContainer(identifier:)` with the dev environment) that round-trips a blob and asserts the record's only payload field is a `CKAsset`.

## 5. Concurrency

- Swift 6 language mode, strict concurrency. No `@unchecked Sendable` without a comment explaining why.
- Repositories are `actor`s over a GRDB `DatabasePool`. Use cases are `Sendable` structs. View models are `@MainActor @Observable`.
- Sync runs on its own actor, triggered by: launch, foreground, 5-minute timer while active, and after each persisted change (coalesced, 2 s).
- Never block the main actor on crypto or network. Decryption of the timeline is paged (50 entries) and cached.

## 6. Testing

- Framework: Swift Testing (`@Test`, `#expect`) for new code; XCTest only where a framework requires it (UI tests).
- Layers:
  - Domain: pure unit tests, fast, no fakes needed.
  - Application: use cases against in-memory fakes from `JourbalTestKit`. Every use case has tests for the happy path and each thrown error.
  - Adapters: contract tests (shared) + adapter-specific tests (migrations apply cleanly from v1 on a fixture DB; cipher round-trips; Keychain adapter against a test access group).
  - Apps: a small number of UI tests for the stated requirements: unlock → cursor at end of today; reflection pane never sends without explicit Ask; "Only my words" filter hides every pin.
- Golden tests for the sync envelope format; changing it is a versioned migration, not an edit.
- Sync: a two-client test over an in-memory `SyncTransport` fake that exercises every branch of the conflict rule; the CloudKit adapter has one round-trip integration test (dev container) that CI runs on a schedule, not per PR.
- A change is not done until `scripts/test` passes on both platforms.

## 7. Tooling & commands

```
scripts/bootstrap     # installs swiftlint, swiftformat, xcodegen; resolves packages
scripts/build         # xcodebuild -workspace Jourbal.xcworkspace -scheme JourbalMac -destination 'platform=macOS'
scripts/test          # swift test for packages, xcodebuild test for both app targets
scripts/lint          # swiftlint --strict, swiftformat --lint
```

- SwiftFormat and SwiftLint configs are in the repo root; do not disable rules inline without a reason in the comment.
- Xcode project is generated from `project.yml` (XcodeGen) so diffs are reviewable. Never hand-edit `.pbxproj`; edit `project.yml` and regenerate.
- Secrets (the Claude API key) are never in the repo. They are entered in Settings and stored in Keychain.

## 8. Working conventions for agents

- **Read before writing.** Open the relevant port and its contract test before implementing an adapter; open the use case before touching a view model.
- **One layer per change** where possible. A PR that adds a column, a port method, a use case and a view is four PRs.
- **Ports first.** When a feature needs a capability the app lacks, add the port and an in-memory fake, write the use case against them, then the adapter.
- **No crypto invention.** Key derivation, encryption, nonce handling: use the documented library call. If a task seems to need something custom, stop and write an ADR instead.
- **Nothing leaves the device by accident.** Any code path that performs a network call to the LLM must go through `AskReflection`, which requires an explicit `ReflectionScope` the user chose. Add a test asserting the provider is never invoked without one.
- **Health is iOS only.** Code under `#if canImport(HealthKit)`; the Mac uses the mirrored tables. Do not add a Mac Health code path.
- **ADR for anything structural.** New adapter, new endpoint, schema change, new dependency: `docs/adr/NNNN-title.md` with context, decision, consequences. Keep them short.
- **Dependencies:** GRDB, swift-sodium (or CryptoKit where sufficient), XcodeGen. Adding anything else needs an ADR.
- **Commits:** imperative subject, ≤ 72 chars, body explains why. Reference the ADR or DESIGN.md section when relevant.
- **Definition of done:** tests at the right layer, lint clean, both platforms build, `DESIGN.md` updated if behaviour changed, ADR if structure changed.
- **When unsure,** prefer the smaller, more native, more boring option and say so in the PR description.

## 9. Things not to do

- No SwiftData for the store (migration and encryption story is too opaque); no Core Data.
- No Combine in new code; async/await and `@Observable`.
- No third-party UI frameworks, no web views.
- No background sync on iOS beyond `BGAppRefreshTask`; no silent push.
- No server of our own in v1, and no CloudKit field that holds plaintext — not even "just for search". Search is local.
- No `NSPersistentCloudKitContainer` or SwiftData CloudKit sync; CloudKit is only ever a blob store behind the port.
- No feature flags, A/B hooks or telemetry. Single user.
