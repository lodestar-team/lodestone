# lodestone

> The ore the fleet's instruments are born from.

A Claude Code plugin that **forges a complete Graph Horizon data service** from a short
spec. Every Horizon data service is the same two layers — an on-chain `DataService`
contract (reusing the shared `GraphTallyCollector`) and an off-chain Rust gateway that
validates TAP receipts, serves/proxies data, and collects fees. Lodestone distils that
pattern, learned from ~10 hand-built services, into a one-interview generator.

## What it generates

- **Contract** — `<Name>DataService.sol` + interface (UUPS upgradeable, parameterised
  tiers/economics), a deploy script, and a Foundry test that deploys against a mock
  Horizon stack and exercises the provider lifecycle.
- **Gateway** — built on [`horizon-core`](https://github.com/lodestar-team/horizon-core).
- **Everything around it** — `gateway.toml`, `docker-compose.yml`, `Dockerfile`,
  `foundry.toml`, a contract-deps setup script, `.env.example`, `yatr.toml`, README.

## Two archetypes

- **`proxy`** — front an existing upstream HTTP data plane (RPC, REST, files, graph-node).
  The gateway is `horizon-core`'s `run()` verbatim. (FHSCE, compass, wsaas, drpc, camp.)
- **`pipeline`** — the service *is* the indexer: `Substrate → Handler → Sink`, with a
  `horizon-core` gateway fronting the query layer. (seahorn.)

## Usage

Invoke the skill in Claude Code:

```
/create-data-service
```

…or just ask: *"make a new Horizon data service for X"*. The skill interviews you,
writes an `answers.json`, and runs the generator:

```bash
python3 skills/create-data-service/scaffold.py --answers answers.json --out ./my-service
```

It then vendors the contract libs and verifies the result builds (`forge build && forge
test`, `cargo build`).

## Install

Add the lodestar-team marketplace, then install:

```
/plugin marketplace add lodestar-team/lodestone
/plugin install lodestone
```

Or point Claude Code at a local clone for development.

## Layout

```
.claude-plugin/plugin.json
skills/create-data-service/
  SKILL.md                  the interview → generate → verify flow
  scaffold.py               the generator (token substitution + tier-enum codegen)
  reference/gotchas.md      hard-won Foundry/horizon-core footguns + canonical addresses
  templates/
    common/                 root files (foundry.toml, .env, yatr, setup-contracts.sh)
    contracts/              the Solidity contract, interface, deploy script, test
    proxy/                  proxy-archetype gateway, config, Docker, README
    pipeline/               pipeline-archetype core + indexer + gateway, config, Docker, README
    pricing-overlay/        optional per-endpoint pricing module + run_with gateway main
mcp/                        horizon-ds-mcp — MCP server for OPERATING deployed services
```

## Operating what you generate — `horizon-ds-mcp`

The [`mcp/`](mcp/) directory is a companion Model Context Protocol server for *operating*
deployed data services: read a contract's economics, a provider's status, a consumer's
escrow balance, on-chain `tokensCollected`, and gateway health — plus an opt-in on-chain
provider lifecycle (register / start / stop). Read-only by default. See [mcp/README.md](mcp/README.md).

Apache-2.0. Experimental community tooling; not affiliated with The Graph Foundation.
