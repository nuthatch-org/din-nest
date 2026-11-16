# din-nest

A [nuthatch](https://github.com/nuthatch-org/nuthatch) nest for the DIN protocol contracts, built as a
comparison spike for [InfiniteZeroFoundation/DevNet discussion 150](https://github.com/InfiniteZeroFoundation/DevNet/discussions/150).
It runs against a local anvil deploy of `develop` (c871ed1), not the dropped DevNet 1.0 contracts on OP
Sepolia, and it is measured against PR 29's subgraph (613f5b2).

One binary, no Docker or Postgres. Events land in local SQL; `views/10-din.sql` holds the derived
entities.

## Run it

```sh
# in DevNet/foundry: the local chain, then the platform plus one model taken to GI 1 local-model submissions
./anvil.sh --host 127.0.0.1
cp /path/to/din-nest/spike/NuthatchSpike.s.sol script/
forge clean && forge script script/NuthatchSpike.s.sol --sig "spike()" \
  --rpc-url http://127.0.0.1:8545 --broadcast --unlocked \
  --sender 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

# here
nuthatch dev --dir . --listen 127.0.0.1:8289 --state-rpc http://127.0.0.1:8545
nuthatch sql --url http://127.0.0.1:8289 "SELECT * FROM global_iterations"
```

The addresses in `nuthatch.toml` are the ones a fresh anvil gives that deploy. A different deploy
means editing them, as with the subgraph's `update-subgraph-addresses.sh`.

## What it covers, against PR 29

I read every handler in PR 29 and sorted its 75 entities by what SQL needs to reproduce them:

| | entities | how |
|---|---|---|
| a raw event row | 36 | the event table, columns renamed |
| a view over events | 34 | latest per key, running sums, first-seen, joins |
| a contract read at the event's block | 5 | a row-driven `eth_call`, then a view |
| logic SQL cannot express | 0 | |

The five reads are the registry getters the subgraph calls (`modelRequests`, `getModel`,
`manifestRequests`, the four fee getters, `feeRouter`). `getModel` is wired here as an example
(`model_record`, sliced by the `model_contracts` view); the other four are the same shape and are not
written yet.

`Validator` was the nearest thing to handler logic. Its active stake floors slashes at zero step by step,
which has a closed form: the final running sum less its lowest negative point. The `validators` view
uses that. No slash happens in the spike's run, so the floor itself is untested.

Five views are written and checked against the run: `models`, `model_contracts`, `validators`,
`global_iterations` and `registrations`. They cover four of the 34 view-shaped entities (`Validator`,
`GlobalIteration` and both registrations); the other 30 are not written. This is a spike, not a port.

## Finding the task contracts

Every task contract is authorised as a slasher before a model can be registered, and `DinCoordinator`
emits `SlasherContractAdded(address)` when it is. The nest's factory reads that event, so no contract
change is needed. Both task contracts were found at blocks 46 and 47 of the run. Which model a pair
belongs to comes from `getModel` (`model_contracts`).

Two things lose here. A nuthatch child takes one template, so the coordinator and auditor ABIs are
merged into one (`abis/DINTask.json`) and their tables share the `task__` prefix; the `address` column
says which contract a row came from. And events a task contract emits before it is announced are not
indexed: 5 in this run (ownership and GI setup states at blocks 41 to 45). The subgraph starts its
templates later still, at `ModelApproved`, so it misses those too.

## Measured on the spike's run

Chain at block 844, 137 logs from the nine contracts. The nest holds 118 of them. The other 19 are the
proxies' own `Upgraded` and `AdminChanged` (2 on each of the 7 platform proxies, which the subgraph
does not index either) and the 5 pre-announcement events above. Every view's rows match what the
script did: 1 model, 9 validators at 10 DIN, GI 1 in state 10, 6 aggregator and 3 auditor
registrations. Backfill took under a second; at this size it measures nothing.

## The read API

`GET /sql?q=…` or `POST /sql` with `{"q": "…"}` answers JSON, or Arrow with
`Accept: application/vnd.apache.arrow.stream`. Each answer carries its provenance (the block it is as
of, how much is sealed). `views/*.sql` are also named queries at `/q/{name}`. So dincli would send SQL,
or call a named query, where it now sends GraphQL. There is also an MCP server (`nuthatch mcp`).

It is a static binary with no runtime dependencies, so it can run inside the din-node container next
to `dind`. I have not tried that container.

## Reorgs and finality

Rows arrive in a mutable hot store. A reorg rolls back the rows of the replaced blocks, and views read
hot plus sealed, so the next query no longer sees them. Only blocks past finality are sealed into
immutable Parquet segments. OP mainnet is in nuthatch's chain registry with finality from the
`finalized` tag. OP Sepolia is not, so today it takes a fixed `finality_depth` in `mounts.toml`;
adding it to the registry with the `finalized` tag is a small change I can make.

## Where it loses to the subgraph

- No handlers. The 34 view-shaped entities have to be written as SQL; none needs more, but it is work.
- Not GraphQL. dincli's reads would change.
- The merged task template, and the 5 pre-announcement events, as above.
- The four other registry reads are not wired yet.
