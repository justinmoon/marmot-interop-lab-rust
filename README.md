# marmot-interop-lab-rust

Phased plan for a Rust-based Marmot interop harness.

- Phase 1: `PLAN.md` (Rust <-> Rust over local Docker relay)
- Phase 2: `OPENCLAW-INTEGRATION-PLAN.md` (Rust harness <-> deterministic Rust bot process)
- Phase 3: `OPENCLAW-CHANNEL-DESIGN.md` + `rust_harness daemon` (JSONL sidecar integration surface)
- Phase 4: Local OpenClaw gateway E2E: Rust harness <-> OpenClaw `marmot` channel (Rust sidecar spawned by OpenClaw)

## Run Phase 1

```sh
./scripts/phase1.sh
```

Defaults:
- Relay URL: random free localhost port (discovered via `docker compose port`)
- State dir: `.state/` (reset each run by the script)

## Run Phase 2

```sh
./scripts/phase2.sh
```

## Run Phase 3 (Daemon JSONL Smoke)

```sh
./scripts/phase3.sh
```

## Run Phase 4 (OpenClaw Marmot Plugin E2E)

This uses the pinned OpenClaw checkout under `./openclaw/`, runs a local relay on a random port,
starts OpenClaw gateway with the `marmot` plugin enabled, then runs a strict Rust harness invite+reply
scenario against the plugin's pubkey.

```sh
./scripts/phase4_openclaw_marmot.sh
```
