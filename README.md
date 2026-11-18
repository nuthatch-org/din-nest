# din-nest

A [nuthatch](https://github.com/nuthatch-org/nuthatch) nest for the DIN protocol contracts, built as a
comparison spike for [InfiniteZeroFoundation/DevNet discussion 150](https://github.com/InfiniteZeroFoundation/DevNet/discussions/150).
It runs against a local anvil deploy of `develop` (c871ed1), not the dropped DevNet 1.0 contracts on OP
Sepolia, and it is measured against PR 29's subgraph (613f5b2).

One binary, no Docker or Postgres. Events land in local SQL; `views/*.sql` hold the derived entities.

## Run it

```sh
# in DevNet/foundry: the local chain
./anvil.sh --host 127.0.0.1

# in another shell: the platform, one model taken through GI 1 to GIended, then the owner-side
# governance calls (three forge scripts in spike/; it runs forge clean first)
DEVNET=/path/to/DevNet spike/run.sh

# here
nuthatch dev --dir . --listen 127.0.0.1:8289 --state-rpc http://127.0.0.1:8545
nuthatch sql --url http://127.0.0.1:8289 "SELECT * FROM global_iterations"
python3 spike/check.py        # every view against what the scripts did, and the chain where it answers
```

The first script, `NuthatchSpike.s.sol`, is the original run: it stops at GI 1's local-model
submissions. `NuthatchGI.s.sol` takes GI 1 on to GIended in three stages with blocks mined between
(the seed locks wait `disputeSeedDelay` blocks), and `NuthatchGov.s.sol` makes the owner calls the
first two leave out: fee, stake and S5/S6 settings, model stake bounds, the fee router's sources and
splits, a manifest update, a rejected request, an unstake, a slasher removal. Without them most of
the views below have no rows to be checked against.

The addresses in `nuthatch.toml` are the ones a fresh anvil gives that deploy. A different deploy
means editing them, as with the subgraph's `update-subgraph-addresses.sh`. The `[[calls]]` read
state at the event's block, so `--state-rpc` has to point at a node that keeps it (anvil does).

## What it covers, against PR 29

I read every handler in PR 29 and sorted its 75 entities by what SQL needs to reproduce them:

| | entities | how |
|---|---|---|
| a raw event row | 36 | the event table, columns renamed |
| a view over events | 34 | latest per key, running sums, first-seen, joins |
| a contract read at the event's block | 5 | a row-driven `eth_call`, then a view |
| logic SQL cannot express | 0 | |

The 34 are the entities a handler loads, changes and saves again, or keys by a natural id so that a
later event replaces an earlier one. The first count came without a list; the list is below, so it
can be argued with.

### The 34 view-shaped entities

**32 are written and match the run. 2 are written and have no rows on it, so they are not checked.**
`nuthatch sql` runs all 34. `python3 spike/check.py` is the check: 55 assertions, all pass, and it
prints the two it cannot check.

| entity | view | checked on the run | note |
|---|---|---|---|
| Validator | `validators` | yes, with gaps | no `pending_withdrawal`, `withdraw_available_at` or `status` (below) |
| StakeGovernance | `stake_governance` | yes | |
| ModelStakeBounds | `model_stake_bounds` | yes | two calls for model 0, the second stands |
| SlasherRegistration | `slasher_registrations` | yes | one row per (slasher, source contract); removal shown |
| S5GlobalParams | `s5_global_params` | yes | |
| S5RecidivismParams | `s5_recidivism_params` | yes | |
| S6Params | `s6_params` | yes | |
| UnstakeRequest | `unstake_requests` | yes | `claimed` is always false, as PR 29 writes it |
| FeeSource | `fee_sources` | yes | one added, one removed |
| FeeRouterConfig | `fee_router_config` | yes | both splits are JSON tuples, sliced with `json_extract_string` |
| GlobalIteration | `global_iterations` | yes, with differences | below |
| AggregatorRegistration, AuditorRegistration | `registrations` | yes | |
| GIStateTransition | `gi_state_transitions` | yes | GI 1 enters states 5 to 28 once each, in order |
| Tier1Batch | `tier1_batches` | yes | |
| Tier2Batch | `tier2_batches` | yes | |
| T1AggregationSubmission, T2AggregationSubmission | `t1_`, `t2_aggregation_submissions` | yes | |
| T1AggregationCommitment, T2AggregationCommitment | `t1_`, `t2_aggregation_commitments` | yes | |
| SeedLock | `seed_locks` | audit and aggregation kinds only | the dispute kind and every reanchor have no rows |
| AggregationDispute | `aggregation_disputes` | **no rows** | the status state machine is written from the handlers and has never run |
| TaskCoordinatorGovernance | `task_coordinator_governance` | yes | |
| LocalModelSubmission | `local_model_submissions` | yes | `eligible` filled by EligibilityFinalized |
| AuditBatch | `audit_batches` | yes | `pending_reassignment` is never true on the run |
| AuditScoreSubmission | `audit_score_submissions` | yes | |
| AuditScoreCommitment | `audit_score_commitments` | yes | revealed, score 80, for all nine |
| EligibilityVote | `eligibility_votes` | yes | |
| EligibilityResult | `eligibility_results` | yes | |
| TestDataCommitment | `test_data_commitments` | yes | |
| EncryptedTestDataKeys | `encrypted_test_data_keys` | yes | |
| TestDataDispute | `test_data_disputes` | **no rows** | no dispute is opened on the run |
| RewardSplit | `reward_splits` | yes | JSON tuple |
| TaskAuditorGovernance | `task_auditor_governance` | yes | three settings, each from its own event |

"Checked" means the view's rows equal what the scripts did (counts, ids, values, order) and, for the
state the contracts expose, what `cast call` returns at the head. It does not mean compared with a
running subgraph: I did not run graph-node, so no row here was diffed against PR 29's own output.

**Where the written views differ from the subgraph, found while checking:**

- `global_iterations.started_at` is the first `GIStateChanged` of the GI. The subgraph creates the row
  on the first event of any kind for it, which here is `RewardDeposited`, one block earlier (93, not
  94). The view also leaves GI 0 (the setup states) out, where the subgraph has a row for it, and
  carries no `global_model_cid`; `tier2_batches.global_model_cid` has it.
- `validators` has the totals, `active_stake`, `blacklisted`, `jailed`, `jailed_until`,
  `total_withdrawn`, `active_registrations` and `encryption_key`. It has no `pending_withdrawal`,
  `withdraw_available_at` or `status`. The subgraph works `status` out from the minimum stake as it
  stood at that validator's last event, which needs an as-of join I did not write. `active_stake` is
  the closed form described below and ignores how a slash eats a pending withdrawal once active
  stake is gone; no slash happens on the run, so that is untested.
- Rows that need a `LocalModelSubmission` (commitments, submissions, votes, results) are kept only
  when one exists, as the subgraph skips them; the view does not check that it came first.
- `aggregation_disputes` and `test_data_disputes` are written from the handlers and are untested.
  Nothing on the run opens a dispute (it needs a challenger and the second phase of DevNet's PR
  149), so their transitions, the placeholder row the subgraph makes when it sees a settlement with
  no opening, and the dispute seed locks are unverified.

`Validator` was the nearest thing to handler logic. Its active stake floors slashes at zero step by
step, which has a closed form: the final running sum less its lowest negative point. The
`validators` view uses that, and the same form for `active_registrations`.

### The 5 contract reads

The subgraph's handlers read the registry with `modelRequests`, `getModel`, `manifestRequests`, the
four fee getters and `feeRouter`. Each is a row-driven `[[calls]]` in `nuthatch.toml`, pinned to the
block of the event that triggers it, and a view slices the returned words.

| entity | read | view | checked |
|---|---|---|---|
| Model | `getModel` | `model_record`, `model_contracts`, `models`, `current_manifests` | yes |
| ModelRegistrationRequest | `modelRequests(requestId)` | `model_registration_requests` | yes: one approved, one rejected |
| ManifestUpdateRequest | `manifestRequests(requestId)` | `manifest_update_requests` | yes: one approved, one rejected |
| FeeSnapshot | `openSourceFee()`, `proprietaryFee()`, `openSourceUpdateFee()`, `proprietaryUpdateFee()` | `fee_snapshots` | yes: four granular, one atomic |
| RegistryFeeWithdrawal | `feeRouter()` | `registry_fee_withdrawals` | yes |

Two things about how they are wired. `on_any` needs a `graph` build of nuthatch, so the four fee
getters are declared once per setter event, 16 small `fee_*_on_*` tables, rather than once across the
four. And the SQL surface has no hex-to-number function, so a `uint256` word in a call result is
turned into a `DECIMAL(38,0)` by an expression over its 32 low hex digits, written out in
`views/40-din-reads.sql`; `spike/gen-reads.py` writes that file. It is right for any value under
10^38 and wrong above, which no fee here comes near.

`ManifestUpdateRequest.requester` is the model's owner, as the subgraph derives it. The subgraph falls
back to the transaction's sender when the model is unknown; the nest does not keep senders, so that
case would be NULL. The contract only accepts a request from the owner of a model, so it should not
arise.

## What cannot be reproduced

Nothing in PR 29's schema needed logic SQL cannot express. These are the places where the nest does
not give what the subgraph gives, or where I could not check that it does:

- **Events a task contract emits before it is announced.** 5 on the run, at blocks 52 to 56, before
  `SlasherContractAdded` at 57 and 58. The nest finds task contracts from that event, so it starts
  there. The subgraph starts its templates at `ModelApproved`, which is later still, so it misses
  them too: a loss against the chain, not against the subgraph.
- **One template for two contracts.** A nuthatch child takes one template, so the coordinator and
  auditor ABIs are merged (`abis/DINTask.json`) and their tables share the `task__` prefix; the
  `address` column says which contract a row came from. Views that need the pair read
  `model_contracts`.
- **The gaps listed above:** `validators.pending_withdrawal`, `withdraw_available_at` and `status`;
  GI 0; `global_iterations.started_at`; the two dispute views and the dispute seed locks, which are
  written but have never seen a row.
- **A diff against a running subgraph.** Not done. I did not stand up graph-node and Postgres.

## Answers to umeradl's constraints

[The comment](https://github.com/InfiniteZeroFoundation/DevNet/discussions/150#discussioncomment-19637768),
point by point.

1. **Not the current OP Sepolia contracts.** The nest targets a local anvil (`chain_id = 1337`) with
   the contracts from `forge script` on `develop` at c871ed1; nothing in it names OP Sepolia or the
   DevNet 1.0 addresses. For DevNet 2.0 when issue 284 lands, the edit is `rpc_urls`, `chain`,
   `chain_id`, and the addresses and `start_block`s in `nuthatch.toml`, from
   `foundry/deployments/<network>.json`.
2. **Task contracts are per model.** Found from `DinCoordinator`'s `SlasherContractAdded`, which every
   task contract triggers before its model can be registered. No contract change. See "Finding the
   task contracts".
3. **Coverage yardstick: PR 29's `schema.graphql`.** The two tables above: 34 view-shaped entities,
   32 written and matching the run and 2 written and untested; 5 reads, all 5 matching. The other 36
   of the 75 are raw event tables and are not reviewed here.
4. **Consumer side.** `dincli` would send SQL or call a named query where it now sends GraphQL; see
   "The read API". I have not tried it in the din-node container. The nest is one static binary, so it
   needs nothing the container lacks, but `nuthatch dev` is a long-running process of its own beside
   `dind`, and I have not looked at how the container would start it.
5. **Reorgs and finality.** See "Reorgs and finality". Not exercised on this run: nothing here forks
   the chain.

## Finding the task contracts

Every task contract is authorised as a slasher before a model can be registered, and `DinCoordinator`
emits `SlasherContractAdded(address)` when it is. The nest's factory reads that event, so no contract
change is needed. Both task contracts were found at blocks 57 and 58 of the run. Which model a pair
belongs to comes from `getModel` (`model_contracts`).

## Measured on the run

Chain at block 240 when indexed, 269 logs. The nest holds 236 of them as event rows, plus the call
rows. The other 33 are the 14 `Upgraded` and `AdminChanged` on the 7 platform proxies (which the
subgraph does not index either), 14 from the implementations and proxy admins (an `Initialized` on
each of 7 implementations, an `OwnershipTransferred` on each of 7 proxy admins), and the 5
pre-announcement events above. Backfill took under a second; at this size it measures nothing.

## The read API

`GET /sql?q=…` or `POST /sql` with `{"q": "…"}` answers JSON, or Arrow with
`Accept: application/vnd.apache.arrow.stream`. Each answer carries its provenance (the block it is as
of, how much is sealed). `views/*.sql` are also named queries at `/q/{name}`. So dincli would send SQL,
or call a named query, where it now sends GraphQL. There is also an MCP server (`nuthatch mcp`).

## Reorgs and finality

Rows arrive in a mutable hot store. A reorg rolls back the rows of the replaced blocks, and views read
hot plus sealed, so the next query no longer sees them. Only blocks past finality are sealed into
immutable Parquet segments. OP mainnet is in nuthatch's chain registry with finality from the
`finalized` tag. OP Sepolia is not, so today it takes a fixed `finality_depth` in `mounts.toml`;
adding it to the registry with the `finalized` tag is a small change I can make.

## Where it loses to the subgraph

- No handlers. The 34 view-shaped entities are SQL; none needs more, but it is work, and two of the
  34 are not yet tested.
- Not GraphQL. dincli's reads would change.
- The merged task template, and the 5 pre-announcement events, as above.
- The gaps in `validators` and `global_iterations`, as above.
