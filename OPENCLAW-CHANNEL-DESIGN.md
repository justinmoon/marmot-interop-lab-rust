# Marmot Channel Design For OpenClaw (Rust-First)

This repo’s goal is a **Rust implementation of Marmot** with **minimal TypeScript glue** for OpenClaw.

OpenClaw already has mature channel implementations (e.g. Telegram, Signal) and a mature channel plugin interface (e.g. Matrix plugin). The Marmot channel should copy the same *shape* so it can grow features over time, even if Phase 1 only supports plain text.

## Goals

- A Rust “engine” that owns all Marmot protocol details:
  - Nostr relay connectivity
  - MLS state, welcomes, commits, group messages
  - persistence and replay/sync
- A small OpenClaw plugin that looks like other channels:
  - `monitor` loop (inbound)
  - `send` (outbound)
  - `probe` / healthcheck
  - onboarding and configuration
  - allowlist/pairing policy hooks

## Non-Goals (Initial)

- Full UI parity with Telegram/Signal immediately (reactions, edits, media, threads).
- Multi-device key sync across OpenClaw installations.

## Architecture Overview

Pieces:

- **Rust sidecar daemon** (this repo):
  - binary: evolve `rust_harness bot` into a long-running process (call it `marmotd` eventually)
  - communicates over **stdio JSON lines** (one JSON object per line)
  - keeps state under a configured `state_dir`

- **OpenClaw channel plugin** (minimal TS, in OpenClaw repo later):
  - spawns the sidecar, restarts it on crash
  - translates OpenClaw inbound/outbound message representations to/from sidecar JSON messages
  - implements OpenClaw channel plugin hooks: config, pairing, allowlists, directory, etc.

This mirrors how channels like Telegram/Signal are structured conceptually:

- “monitor provider” produces inbound messages into the router
- “send” implements delivery surface rules + attachments formatting
- “probe” / health check validates connectivity
- account resolution + security policy is centralized

## Sidecar Protocol (JSONL)

### Principles

- Stable, explicit versioning (`protocol_version`)
- Deterministic behavior (idempotent commands where possible)
- No implicit global state: everything under `state_dir`

### Outbound From Sidecar (stdout)

Events:

- `ready`:
  - fields: `pubkey`, `npub`, `protocol_version`
- `log`:
  - fields: `level`, `msg`, `fields`
- `keypackage_published`:
  - fields: `event_id`, `created_at`
- `welcome_received`:
  - fields: `giftwrap_id`, `welcome_id`, `from_pubkey`
- `group_joined`:
  - fields: `nostr_group_id`, `mls_group_id`
- `message_received`:
  - fields: `nostr_group_id`, `from_pubkey`, `content`, `created_at`, `message_id`

### Inbound To Sidecar (stdin)

Commands:

- `publish_keypackage`:
  - fields: `relays`
- `set_relays`:
  - fields: `relays`
- `accept_next_welcome`:
  - fields: optional filters (e.g. `expected_from_pubkey`)
- `send_message`:
  - fields: `nostr_group_id`, `content`
- `shutdown`

All commands should respond with either:

- `ok` with a `request_id`, or
- `error` with `request_id`, `code`, `message`

## OpenClaw Mapping

### Targets

OpenClaw “To” fields should map cleanly to Marmot:

- DMs (future): `marmot:<npub>`
- Groups: `marmot:group:<nostr_group_id_hex>`

The OpenClaw plugin should implement a `normalizeTarget(...)` similar to other channels.

### Security Model

Copy the OpenClaw channel conventions:

- DM policy: `pairing` by default
- group policy: `allowlist` by default
- group require-mention gating when group is “open”

### Directory

Phase 1: directory can be minimal:

- list peers from allowlists + recent sessions
- list groups from local state

## Persistence Model

Sidecar owns persistence:

- identity key (nostr)
- MLS state (sqlite or equivalent)
- message index for sync/backfill
- relay list

OpenClaw plugin should treat sidecar state as authoritative and avoid duplicating protocol state.

## Roadmap (Feature Growth)

1. Phase 3a: JSONL protocol + long-running sidecar, OpenClaw plugin can send/receive plain text.
2. Phase 3b: multi-account support (like Telegram/Signal accounts in OpenClaw).
3. Phase 3c: attachments (OpenClaw media pipeline -> sidecar reference, or sidecar uploads).
4. Phase 3d: reactions, edits, threads (if Marmot layer supports it).
5. Phase 3e: directory search, group management UI hooks.

