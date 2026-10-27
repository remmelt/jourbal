# Jourbal

Personal journaling app for macOS and iOS. The Mac is for longform writing, the iPhone for check-ins and reading.

- Client-side encryption; CloudKit private database used only as a blob store. No server of our own.
- Mood tracking, synced with Apple Health on iOS.
- Optional reflection via the Claude API. Nothing is sent unless you explicitly ask.
- Single user. No analytics, no telemetry.

## Docs

- [`DESIGN.md`](DESIGN.md): product brief, data model, screens.
- [`AGENTS.md`](AGENTS.md): architecture (hexagonal, Swift packages), conventions, tooling.
- [`design/`](design/): exported mockups.
- [`docs/design-review.md`](docs/design-review.md): open issues, contradictions, build plan.

## Status

Design phase. No code yet.
