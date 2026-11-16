---
name: nuthatch
description: Query this self-hosted nuthatch nest on din-local - decoded events, balances, and read-only SQL. Use when asked about on-chain activity for these contracts.
---

# Querying the nuthatch nest

Contracts indexed on din-local:
- `registry` = 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1
- `stake` = 0xA51c1fc2f0D1a1b8494Ed1FE312d7C3a78Ed91C0
- `coordinator` = 0xa513E6E4b8f2a923D98304ec87F64353C4D5C853
- `token` = 0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
- `fee_router` = 0x5FC8d32690cc91D4c39d9d3abcBD16989F875707
- `treasury` = 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
- `emission` = 0xc6e7DF5E7b4f2A278906862b61205850344D4e7d

Data is local - never call an external API for it.

## Preferred: MCP
If a `nuthatch` MCP server is configured, use its tools. Call `schema` first to learn the
data model, then `sql` / `entity` / `balance` / `top_balances` (read `complete_history` in their results: when false or null the figures are `net_flow_since_indexed_from`, not balances; true means indexed from the configured `start_block` with no `--backfill`, the deployment if `start_block` is the deployment block, which `init` sets).

## Fallback: HTTP (a `nuthatch dev` must be running)
- Recent rows:  `curl localhost:8288/entities?limit=20`
- Read-only SQL: `curl -G localhost:8288/sql --data-urlencode 'q=SELECT count(*) FROM coordinator__din_per_eth_updated'`
- Balances: `GET /balances`, read with the status at `GET /`: the figures are net flow since its `start_block`, balances only when it says `history_from_deployment: true` (false or null means net flow). True means indexed from the configured `start_block` with no `--backfill`: the deployment only if `start_block` is the deployment block, which `init` sets

`sql` reads hot and sealed rows together, so it covers the live tip. Each answer's
`provenance.sealed_through` is the block through which this nest has sealed history; newer rows
may still be hot, and it is not the chain's finality head.
