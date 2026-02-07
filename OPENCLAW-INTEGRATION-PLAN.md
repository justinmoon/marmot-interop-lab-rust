# Marmot Interop Lab (Rust Track) - Phase 2 Plan (With OpenClaw)

This document starts **only after** `marmot-interop-lab-rust/PLAN.md` is implemented and the local-relay **Rust A <-> Rust B** scenario passes reliably.

Note: this repo intentionally avoids any TypeScript Marmot SDK implementation. Phase 2 is built around a deterministic **Rust bot process** (this repo) that OpenClaw can later integrate with by spawning it as a sidecar.

## Goal

Add an automated local-relay test proving:

- **Client A = Rust harness** can invite **Client B = deterministic Rust bot process** and exchange two-way MLS application messages (E2EE over Nostr group events).
- The test is repeatable, deterministic, and uses only local state + local relays.

In Phase 2 we still avoid any LLM dependency. The bot must be deterministic and reply in-code, not via model inference.

## Non-Goals (For Phase 2)

- Public relays, NIP-65 outbox routing, relay discovery.
- Shipping/publishing an extension in the first pass (but we design for it).
- Solving all cross-implementation serialization issues up front (we will isolate and document them when encountered).

## Inputs / Prior Art (Read, Then Reuse Patterns)

These known-good references exist on this machine and should guide implementation (port patterns, don’t reinvent):

- OpenClaw process management patterns (how extensions spawn sidecars, isolate state, log stdout/stderr).
- Existing JS track (this repo root): deterministic local-relay E2E patterns and “no false positive” assertions.

If cross-implementation pitfalls show up, mine (do not pre-optimize):
- `~/code/whitenoise-emulator/E2E_FRONTENDS_COMPARISON.md`

## Environment / Repro Guardrails

- Do not modify or restart any currently-running OpenClaw containers or `~/.openclaw-*` state.
- Use isolated state dirs under:
  - `marmot-interop-lab-rust/.state/` (Rust)
  - `marmot-interop-lab-rust/.state/openclaw-bot/` (bot fixture)
- Do not depend on public relays; only `ws://localhost:...`.
- Record the exact OpenClaw SHA used in `marmot-interop-lab-rust/OPENCLAW_SHA.txt`.

## OpenClaw Baseline (Pin It) (Future)

We must start from OpenClaw `origin/main` at a pinned SHA (not a dirty local checkout).

Process:
1. `git fetch origin` in `~/code/openclaw`
2. Create a worktree at the pinned SHA (recommended):
   - `git worktree add ~/code/openclaw-interop-rust <SHA>`
3. Do all OpenClaw edits in that worktree only.
4. Write `<SHA>` into `marmot-interop-lab-rust/OPENCLAW_SHA.txt`.

## Phase 2 Success Criteria (Mechanical)

One command from `~/code/marmot-interop-lab/` (or from this folder) exits 0 only if:

1. Local relay is up and reachable (`ws://127.0.0.1:8080` or configured port).
2. Bot fixture starts with isolated state, publishes a KeyPackage (kind `443`), and listens for welcomes (giftwrap kind `1059`).
3. Rust harness invites the OpenClaw fixture by selecting its kind `443` event.
4. OpenClaw fixture joins from the welcome and replies with **exactly** a tokenized string.
5. Rust harness receives and decrypts that reply and asserts:
   - sender pubkey == OpenClaw bot pubkey
   - plaintext == expected reply (byte-for-byte)

No “contains” matching. No “any application message” matching. Avoid false positives.

## Architecture (Minimal)

- Relay: reuse the same local Docker relay(s) as Phase 1.
- Client A: Rust harness (this folder’s Rust code).
- Client B: deterministic Rust bot process (`rust_harness bot`) with isolated state.

The Rust harness must be able to:
- fetch KeyPackage events for a target pubkey (kind `443`)
- create/invite/join (or invite and then wait for reply if OpenClaw handles join)
- send application messages and decrypt inbound ones

## Implementation Plan (Phase 2)

### Step 1: Rust Bot Fixture (In This Repo)

Implement a deterministic bot process in Rust:

- Command: `cargo run -p rust_harness -- bot ...`
- On start, print one parseable line:
  - `[openclaw_bot] ready pubkey=<hex> npub=<npub>`
- Publish a KeyPackage event (kind `443`).
- Listen for welcomes (giftwrap kind `1059`), join the group, then respond to:
  - `openclaw: reply exactly "E2E_OK_<token>"`
  with:
  - `E2E_OK_<token>`

### Step 2: Rust Harness Scenario “Invite OpenClaw And Assert Reply”

Add a Rust scenario that:
1. waits for relay readiness
2. launches the Rust bot process and captures the bot pubkey from stdout
3. fetches the bot’s KeyPackage event (kind `443`, author=bot pubkey)
4. invites the bot, waits for join/welcome semantics as needed
5. sends the prompt: `openclaw: reply exactly "E2E_OK_<token>"`
6. waits for a decrypted reply and asserts exact match

Note:
- If the Rust implementation must choose an MLS ciphersuite based on the recipient KeyPackage, do so (avoid "mismatched ciphersuite" failures).

### Step 3: Orchestration Command

Provide one command (script or `cargo` runner) that:
- resets Rust + bot fixture state under `marmot-interop-lab-rust/.state/`
- starts relay(s) (or asserts they are running)
- starts bot fixture
- runs Rust scenario
- always tears down the fixture process (even on failure)

This is analogous to the JS track’s `pnpm test:openclaw-e2e`.

## Interop Pitfalls (Handle When Encountered)

Cross-implementation failures are expected in Phase 2. Requirements for making them debuggable:

- Print the exact kinds and event ids observed/published (443/444/445/1059).
- Log (at least) the MLS ciphersuite id chosen on each side.
- If message decode fails, include a short error with a stable label (e.g. `MLS_DECODE_FAIL`, `NIP44_DECRYPT_FAIL`).
- Keep a "known issues" section in this file once real failures are found.

Do not relax success checks to "make it pass".

## Known Interop Issues (Future)

When integrating this Rust bot into OpenClaw, we expect typical cross-impl pitfalls around:

- child-process lifecycle (restart, state dir handling, logs)
- data model differences (OpenClaw UI expectations vs sidecar state)

## Distribution (Future)

Eventually we want users to install the OpenClaw extension easily.

Design constraints now (even if not implemented yet):
- avoid requiring local git checkouts for end users
- keep the bot fixture test in CI so extension changes don’t silently break interop
- pin and document versions (OpenClaw SHA, Rust crate versions, relay image digest)

Future packaging options to evaluate:
- publish OpenClaw extension to npm and install via OpenClaw’s extension mechanism
- bundle the extension into OpenClaw releases
