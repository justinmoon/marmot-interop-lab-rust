# Marmot Interop Lab (Rust Track) - Phase 1 Plan (No OpenClaw)

## Goal

Create a clean, reproducible, automated end-to-end test that proves **Rust client A can invite Rust client B and exchange two-way MLS application messages**, using a **local Docker Nostr relay**.

This is the baseline sanity check for the Rust implementation. Phase 1 is **Rust <-> Rust only**.

## Constraints / Guardrails

- Do not modify or restart any currently-running OpenClaw containers or `~/.openclaw-*` state.
- Do not depend on public relays.
- Keep all state produced by Rust runs under this folder (do not write into global state dirs).
- Avoid false positives: the scenario must only pass if the expected peer message is actually decrypted and matched.

## Start From Origin (Pin It Immediately)

Phase 1 must be reproducible.

Requirements:
- Commit `rust-toolchain.toml` pinning a specific Rust toolchain version.
- Commit `Cargo.lock`.
- If using any git dependencies (or submodules) for the Rust MLS/Nostr implementation, pin by **exact commit SHA** and record it in files in this folder:
  - `MARMOT_RUST_IMPL_SHA.txt` (or `WHITENOISE_RS_SHA.txt`, if that is the chosen base)
  - `NOSTR_SDK_SHA.txt` (only if using a git dependency instead of crates.io)

Rule: do not use "latest" without writing down the SHA.

## Local Relay (Docker)

Use a local Nostr relay reachable via `ws://...` (not `wss://...`).

Supported options:
1. Single relay (fast path): Strfry in Docker.
2. Full local stack (optional): a multi-relay compose (e.g. `nostr-rs-relay` + strfry) for debugging.

Hard requirement:
- The Phase 1 scenario must include a **relay readiness check** (real WS connect/close loop) before publishing/subscribing to avoid flake.

Pinning requirement:
- Record the relay image digest(s) used in `RELAY_IMAGE.txt` (or a folder-local equivalent) so runs are repeatable.

## Protocol Scope (Match The JS Track Where Possible)

This Rust track is intended to eventually interop with OpenClaw’s `marmot-ts` extension, so we should align on the same Nostr kinds and flow as early as feasible:
- KeyPackage publish: kind `443`
- Welcome rumor: kind `444`
- Group messages / commits: kind `445`
- Giftwrap wrapper: kind `1059`

If the Rust implementation uses different kinds or encodings, document it explicitly and treat Phase 2 as "interop work".

## The Harness (Two Headless Clients)

Implement a Rust harness that can run the full scenario in one command.

Actors:
- **Client A**
  - generate/load Nostr identity key
  - generate/load MLS KeyPackage
  - publish kind `443` KeyPackage event (assert relay ACK or equivalent)
  - create a group (ciphersuite chosen compatibly with B’s KeyPackage, if applicable)
  - invite B by selecting B’s kind `443` event
  - send an MLS application message containing a unique token: `HELLO_FROM_A_<token>`
  - wait for B’s reply `HELLO_FROM_B_<token>` and assert exact match

- **Client B**
  - generate/load Nostr identity key
  - generate/load MLS KeyPackage
  - publish kind `443` KeyPackage event (assert relay ACK or equivalent)
  - poll/subscribe for giftwrap kind `1059`, unwrap and accept Welcome (inner kind `444`)
  - join from Welcome and persist group state
  - subscribe/ingest kind `445` events for the group and decrypt A’s message
  - reply with exact `HELLO_FROM_B_<token>`

### Avoiding False Positives (Required)

The scenario must only pass if it observes a decrypted application message where:
- `rumor.pubkey` (or Rust equivalent) matches the expected peer pubkey
- plaintext content matches **byte-for-byte** expected content (include the unique token)

No "any application message" success conditions.
No "contains token" success conditions.

### Persistence (Required)

Persist state under folder-local paths, e.g.:

```
marmot-interop-lab-rust/.state/a/
marmot-interop-lab-rust/.state/b/
marmot-interop-lab-rust/.state/relay/
```

Persist group state after:
- group create
- invite publish
- join from welcome
- each successful send
- each successful ingest that advances the MLS ratchet

This is for correctness and for post-run debugging.

## Automated Scenario = The Success Criterion

Provide one command that:
1. resets Rust state + relay state
2. brings up relay(s)
3. runs the scenario with deterministic timeouts
4. exits 0 only if both directions succeed

Suggested interface (exact names flexible):
- `cargo run -p rust_harness -- scenario invite-and-chat`
- `cargo test -p rust_harness_e2e -- --nocapture`

## Logging / Observability Requirements

Every run must print:
- relay URL(s) and readiness result
- each published event kind + id for 443/444/445 (and 1059 when observed)
- group id
- for each inbound kind `445`: decrypt success/failure, MLS decode success/failure
- the application plaintext content at the end (short tokenized strings only)

Logs must be information-dense and deterministic (no huge debug dumps by default).

## Timing Notes (Required)

- Giftwrap timestamps are randomized; when scanning for kind `1059`, use a wide lookback window (hours, not seconds).
- Don’t use `since = now-2s` style filters for welcomes; it will miss events.
- For message assertions, include a unique token per run.

## Phase 1 Exit Criteria Checklist

- Client B publishes kind `443` KeyPackage (and we see relay ACK).
- Client A fetches B’s kind `443` and successfully invites.
- Client B observes giftwrap kind `1059`, unwraps, asserts inner kind `444`, joins group.
- Both sides ingest group backlog (kind `445`, `#h=<groupId>` or equivalent).
- A -> B application message decrypts and matches expected.
- B -> A application message decrypts and matches expected.

