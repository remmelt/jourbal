# Design review — 2026-10-27

Review of `DESIGN.md` and `AGENTS.md` against each other. The mockup PDF was not reviewed.

## Bottlenecks (critical path, decide first)

1. **Key and lock model. This blocks everything.**
   - Keychain won't sync an item that has a biometric access control. "Synced key, guarded by Touch ID" can only mean an app-level `LAContext` prompt in front of a readable key. That is a UI gate, not crypto. Say so in the threat model, or keep a synced master copy plus a biometric-protected local copy on each device.
   - Several features need the key while the app is locked: the iPhone check-in "already unlocked", App Intents (Log mood/mark), the Services menu, the menu bar extra showing today's mood, and the background Health mirror. Either they all require unlock (simplest), or writes go to a write-only inbox encrypted with a public key.
   - **No key recovery.** If iCloud Keychain is reset or switched off, every entry is gone for good. A printed recovery key or a passphrase-wrapped key backup is required.
2. **Local store vs. queries.** Ciphertext columns mean SQL can't do the timeline by date, tag filters, search or Year aggregates. Pick one:
   - plaintext DB protected by FileVault / file protection, encrypted only when syncing; or
   - plaintext metadata columns (dates, mood) with encrypted bodies, and an in-memory search index built at unlock.

   At 2k entries, decrypting everything at unlock takes milliseconds, so the 50-entry decryption paging is premature.
3. **Sync granularity.** Once shipped, the envelope format is fixed (golden tests). Settle it now:
   - One `CKAsset` per row is slow for tiny records like moods and tag links. Use a bytes field, and an asset only when the payload is over ~900 KB.
   - `entry_tag` as its own synced row causes churn. Store tags inside the entry, as the domain's `Entry.tags` already does.
   - `kind` in plaintext leaks how many moods vs. entries there are, and when.
4. **iOS notifications.** iOS keeps at most 64 pending local notifications. So:
   - Jitter means scheduling ~30 days ahead and topping up on each launch.
   - Skip rules ("already checked in", "100+ words today") can't be checked when a notification fires. A check-in done on the Mac can't cancel the iPhone's reminder without background sync.
   - "Deliver to active device" can't be done without a server. Drop it.
5. **TextKit 2 editor.** Biggest UI risk, longest task. Anything that touches `.layoutManager` silently falls back to TextKit 1. `complete(_:)` triggered automatically on `#` is janky. Spike it first.

## What doesn't work as written

- **SwiftData:** DESIGN says "GRDB or SwiftData", AGENTS §9 bans SwiftData. Go with GRDB.
- **Fonts:** DESIGN uses Inter for the sans, AGENTS says the system font and "no third face". Use the system font.
- **Colours:** DESIGN gives hex values for background, panel and text; AGENTS requires semantic colours. Use semantic colours for chrome and hex only for the mood scale and the AI amber.
- **Silent push:** AGENTS §4 wants a CloudKit zone subscription for push-triggered sync, §9 says no silent push.
- **Wrong cipher name:** AGENTS says "secretbox (XChaCha20-Poly1305)", but libsodium's secretbox is XSalsa20 and has no AAD. Use an AEAD that binds `id+version` as AAD, or that a malicious store can swap blobs. CryptoKit `ChaChaPoly` does this and removes the swift-sodium dependency.
- **Window restoration:** `NSWindow.restorationClass` doesn't apply to SwiftUI `WindowGroup` windows. Use `@SceneStorage`.
- **Markdown styling:** AGENTS §3 says `NSTextStorage` attributes, §3b says `NSTextContentStorageDelegate`. Pick one; attributes are simpler.
- **Check-ins:** "Appended under the day" plus `AppendCheckIn` means the iPhone writes into the same entry record the Mac is editing, which is a constant conflict. Make each check-in its own record.
- **Pin offsets:** `range_start/range_end` break as soon as the entry is edited. Store the quoted text and re-anchor, or drop the ranges.
- **Marks as workouts:** `health_workout_type` writes zero-length workouts into Fitness, which pollutes it. Probably cut.
- **Pin vs. "Not kept unless you pin":** `pin.reflection_id` needs a reflection row to exist. Define what gets kept when only a pin is saved.
- **Mood mapping:** there's no rule mapping the 5-step mood to Health's continuous valence and back. Values from other apps need binning.
- **Leftovers:** a "Devices" settings tab and a "Calendar provider" port appear in AGENTS with no design behind them. The mockup filename is also wrong (`jourbal-` vs `journal-canvas.pdf`).

## Missing

- `docs/threat-model.md` (AGENTS references it but it doesn't exist).
- First-run onboarding, a mode for no iCloud, adding a new device, revoking a lost one.
- Day boundary rule: does an entry written at 1 am, or while travelling, belong to yesterday? Can there be more than one write per day?
- Merge rule for duplicate tags and mark kinds created offline on two devices (same name, different ids).
- Name aliasing:
  - how names are detected (`NLTagger` `.nameType`)
  - where the alias map is stored
  - how aliases are reversed in replies.
- What "Write about this" does.
- Search design: there is no Mac search UI, and the iOS Search tab is unspecified.
- Payload schema versioning inside the ciphertext, so an older client keeps fields it doesn't know.
- Restore/import from backup; the plain-text export format.
- Sync error states: signed out of iCloud, quota full, zone deleted.
- Attachments: state explicitly whether they're a non-goal.
- Repo scaffolding: `project.yml`, packages, `scripts/` all don't exist yet.

**Overbuilt for a single-user v1:** App Intents, the Services menu, menu bar extra, Handoff, Year poster printing, the 300 ms first-window signpost target, and four PRs per feature. Push them after v1.

## Splitting into sub-agents

The frozen ports are the contract between agents. Each agent owns one package target, works in its own worktree, and doesn't edit ports. Port changes go back through the coordinator.

| Wave | Agents | Parallel? | Output |
|---|---|---|---|
| 0 | Coordinator + owner | no | ADRs for the 5 bottlenecks above; fix contradictions; scaffold (XcodeGen, packages, scripts, lint) |
| 1 | **Domain+Ports** | 1 agent | Entities, ports, TestKit fakes, contract test skeletons. **Freeze.** |
| 1 | **Editor spike** | alongside | TextKit 2 `NSViewRepresentable`: styling, completion, undo, cursor-after-layout. No domain dependency. |
| 2 | **SQLiteStore**, **Crypto+KeyStore+recovery**, **SyncOnce** (in-memory transport, every conflict branch), **CloudKit transport**, **Schedule+Notifier** | 5 parallel | Adapters passing the contract tests |
| 3 | **Mac shell** (window, sidebar, commands, lock), **Integration** (CompositionRoot) | 2 | Build-order step 1+2 milestone |
| 4 | **iOS check-in+Health**, **Marks**, **Year**, **Reflection** (aliasing → provider → pane → pin) | 4 parallel | Features |
| each PR | **Reviewer** | — | Layer rules, no plaintext leaving the device, tests |

**Needs a human:**
- Apple Developer account, iCloud container, entitlements.
- Real-device checks: the two-device round trip, the CloudKit Dashboard ciphertext check, Touch ID / Face ID feel, notification timing.
