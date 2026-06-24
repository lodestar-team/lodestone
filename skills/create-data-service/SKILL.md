---
name: create-data-service
description: Scaffold a complete Graph Horizon data service — Solidity contract, Rust gateway, config, Docker, deploy script, tests. Use when the user wants to create/bootstrap/start a new Horizon data service, a new paid data service on The Graph, a TAP/GraphTally-monetised service, or asks to "make a data service". Supports a proxy archetype (front an existing upstream HTTP data plane) and a pipeline archetype (custom substrate→handler→sink indexer).
---

# Create a Horizon Data Service

You are scaffolding a new data service for The Graph's Horizon framework. Every such
service is two layers: an **on-chain Solidity contract** (inherits `DataService`, reuses
the shared `GraphTallyCollector` unchanged) and an **off-chain Rust gateway** (validates
TAP receipts → serves/proxies data → aggregates RAVs → calls `collect()`). This skill
generates both, plus config, Docker, deploy script, and tests, then verifies they build.

Read `reference/gotchas.md` before generating — it holds the hard-won Foundry/horizon-core
footguns and the canonical contract addresses. Do not skip it.

## Step 1 — Choose the archetype

Two shapes exist. Pick by where the data comes from:

- **`proxy`** — the service sits in front of an existing upstream HTTP data plane (an
  RPC node, a REST API, a file server, a graph-node). The gateway is `horizon-core`'s
  `run()` verbatim: an 11-line `main.rs` plus a TOML file. **Default. Choose this unless
  the service IS the indexer.** Examples: FHSCE (files), compass (graph-node), wsaas (WS),
  camp (REST), drpc (JSON-RPC).
- **`pipeline`** — the service itself ingests a chain and produces data: a
  substrate (gRPC/firehose/RPC stream) → pure `Handler`s → a `Sink` (Postgres). Payments
  still go through a `horizon-core` gateway sitting in front of the query layer. Example:
  seahorn (Solana decode).

If the user hasn't said, ask. Otherwise infer and state your choice.

## Step 2 — Interview

Collect (ask only for what you can't infer from the user's brief; offer sensible defaults):

| Field | Meaning | Default |
|---|---|---|
| `service_slug` | kebab id, used for crate/db names (`oracle`) | derive from name |
| `service_name` | PascalCase, drives `OracleDataService` | derive from slug |
| `service_title` | human title | "Foo Data Service" |
| `service_description` | one-line purpose | — |
| `archetype` | `proxy` \| `pipeline` | `proxy` |
| `tiers` | unit-of-service tiers (the `DataTier` enum) | `["BASIC", "DECODED", "SQL"]` |
| `min_provision` | min GRT provision per provider (Solidity literal) | `555e18` |
| `burn_cut_ppm` | fees burned, PPM (1% = 10000) | `10000` |
| `data_service_cut_ppm` | fees retained, PPM | `10000` |
| `default_port` | gateway port | `8090` |
| `upstream_url` | (proxy only) upstream data plane URL | `http://127.0.0.1:5678` |
| `network` | `arbitrum_sepolia` (testnet) \| `arbitrum_one` | `arbitrum_sepolia` |

Default to Arbitrum **Sepolia** — new services test on testnet first. Never invent a
private key; the deploy script reads it from env.

## Step 3 — Generate

Write the collected answers to a JSON file and run the bundled generator. From the skill
directory:

```bash
python3 scaffold.py --answers /path/to/answers.json --out /path/to/<service_slug>
```

`answers.json` shape:

```json
{
  "service_slug": "oracle",
  "service_name": "Oracle",
  "service_title": "Oracle Data Service",
  "service_description": "Paid price-oracle reads on Horizon.",
  "archetype": "proxy",
  "tiers": [
    {"name": "BASIC",  "comment": "spot price reads"},
    {"name": "STREAM", "comment": "streaming price updates"}
  ],
  "min_provision": "555e18",
  "burn_cut_ppm": "10000",
  "data_service_cut_ppm": "10000",
  "default_port": "8090",
  "upstream_url": "http://127.0.0.1:5678",
  "network": "arbitrum_sepolia"
}
```

The generator copies the right template subtree (`common/` + `contracts/` + `proxy/` or
`pipeline/`), substitutes `{{TOKENS}}` in file contents and filenames, and generates the
`DataTier` enum from `tiers`. It prints the file tree it wrote.

## Step 4 — Vendor contract dependencies

Run the generated setup script from the repo root:

```bash
./setup-contracts.sh
```

It `forge install`s forge-std, OpenZeppelin upgradeable (v5.6.1), and — critically —
`graphprotocol/contracts` **pinned to the `@graphprotocol/horizon@1.1.0` commit**. Do NOT
let it float to the latest tag: `v6.0.0+` reorganises away from the
`packages/horizon/contracts/` layout the remappings expect. This pin is verified to build
and pass tests. See `reference/gotchas.md` for the full story.

## Step 5 — Verify it builds

This is mandatory. A scaffold that doesn't compile is worse than none.

```bash
# Contract
cd <service_slug> && forge build && forge test

# Gateway (proxy)
cargo build            # in the gateway crate

# Indexer + gateway (pipeline)
cargo build --workspace
```

Fix any breakage before handing back. Common failures are in `reference/gotchas.md`
(remapping roots, `via_ir`, the `IGraphTallyCollector` import path, `deregister` not being
`override`). Report the build result honestly — if `forge test` fails, say so with output.

## Step 6 — Hand off

Tell the user what was generated and the exact next steps:
1. Fill `.env` (PRIVATE_KEY, OWNER, PAUSE_GUARDIAN) and the gateway TOML (addresses, db).
2. Deploy the contract to testnet (`forge script ... --broadcast`), note the proxy address.
3. Put the proxy address into the gateway config's `tap.data_service_address`.
4. `docker compose up` to bring up Postgres + the gateway.
5. (proxy) point `backend.upstream_url` at the real data plane; (pipeline) run the indexer.

Do NOT deploy or push anything yourself unless explicitly asked — generation only.
