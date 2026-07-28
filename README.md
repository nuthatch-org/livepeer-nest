# livepeer nest

A nuthatch nest indexing the **Livepeer protocol on Arbitrum One**, built for feature parity
with the official Livepeer subgraph ([`livepeer/subgraph`](https://github.com/livepeer/subgraph),
deployment `QmX9fNJ4invFEVzJFyQUtaVrawWLBMLkST5QetqqfKc46q` /
`FE63YgkzcpVocxdCEyEYbvjYqEf2kb1A6daMYRxmejYC`).

One binary, one command, live SQL — no Postgres, no Docker, no third-party data API.

```sh
nuthatch dev --dir . --rpc https://your-arbitrum-endpoint.example
nuthatch sql --url http://127.0.0.1:8288 "SELECT * FROM protocol"
```

## Contracts indexed

All eleven data sources the subgraph tracks, with the subgraph's own **implementation** ABIs
vendored under `abis/` (Sourcify only exposes the `ManagerProxy` ABI, which omits every real
event — the classic proxy trap).

| alias | contract | address | start block |
|-------|----------|---------|-------------|
| `controller`      | Controller       | `0xD8E8…6ee4` | 5856334 |
| `bondingmanager`  | BondingManager   | `0x35Bc…3e40` | 5856381 |
| `roundsmanager`   | RoundsManager    | `0xdd6f…c39f` | 5856394 |
| `livepeertoken`   | LivepeerToken    | `0x289b…A839` | 5856156 |
| `minterv1`        | Minter (v1)      | `0x4969…752E` | 5856338 |
| `minter`          | Minter           | `0xc20D…7252` | 6253359 |
| `ticketbroker`    | TicketBroker     | `0xa8bB…e41B` | 5856357 |
| `serviceregistry` | ServiceRegistry  | `0xC92d…7431` | 5860363 |
| `pollcreator`     | PollCreator      | `0x8bb5…32E6` | 5857075 |
| `treasury`        | LivepeerGovernor | `0xcFE4…6aa0` | 139793525 |
| `l2migrator`      | L2Migrator       | `0x148D…2085` | 5864923 |

**Factory:** governance polls are deployed dynamically — `PollCreator.PollCreated(indexed poll, …)`
spawns a `Poll` contract emitting `Vote(indexed voter, uint256 choiceID)`. Declared as a nuthatch
`[[templates]]` + `[[factories]]`; children are discovered at runtime into `poll__vote`.

Raw decoded event tables are named `{alias}__{event}` (36 in total). Event allowlists in
`nuthatch.toml` mirror the subgraph's tracked events exactly.

## Derived views (subgraph-entity parity)

`views/*.sql` reconstruct the subgraph's derived entities from the raw event tables:

| view | subgraph entity | status |
|------|-----------------|--------|
| `round`       | `Round`       | exact (event-derived) |
| `transcoder`  | `Transcoder`  | rounds/cuts/status exact; stake fields omitted (see below) |
| `delegator`   | `Delegator`   | delegate/start-round exact; `bonded_amount` = last-Bond snapshot |
| `protocol`    | `Protocol`    | round + counts exact; `inflation` omitted (see below) |
| `poll`        | `Poll`        | metadata + raw vote tallies |

### The parity boundary — what nuthatch can and can't mirror

nuthatch's core is **deterministic and event-only** (a non-negotiable): views derive from decoded
event data, never from `eth_call`. The Livepeer subgraph's AssemblyScript mappings *do* call the
contracts (e.g. `bondingManager.transcoderTotalStake`, `minter.currentMintableTokens`,
`roundsManager` reads) to enrich entities. Fields sourced that way are **structurally outside** a
pure event view:

- `Protocol.inflation` / `inflationChange` / `totalActiveStake`
- `Transcoder.totalStake` / `totalVolumeETH` / `totalVolumeUSD`
- `Poll` stake-weighted outcome (needs bonded stake at poll-end)

Everything derivable from event data (rounds, activation/deactivation, reward cut / fee share,
bond flows, vote counts, all the raw event history) *is* reproduced. The call-derived fields would
need an **effectful WASM annotator** (nuthatch's transform layer escape hatch), which produces
annotations, never canonical entities — a clean follow-up, not a core change.

## Footprint

Full-history backfill from the deployment block (5,856,156) to the Arbitrum tip (~486.6M):

| metric | value |
|--------|-------|
| decoded events | 838,502 across 36 tables |
| sealed Parquet | 164 MB |
| RSS (steady) | well within the 2 GB per-cursor budget (~300 MB observed for peer nests) |
| backfill wall-clock | 41m47s, 334 ev/s, `--seal-direct` on a single RPC (factory nests seal sequentially) |

Live snapshot at round 4276: 233 transcoders (100 active), 5,036 delegators, 1,811 rounds, 7 polls.

Small and cheap to run — comfortably co-tenantable next to other arbitrum-one nests on a modest
8 GB VPS.

## Verification

Validated against **on-chain ground truth** (the source the subgraph itself reads) via `cast` on
Arbitrum One:

| field | nest view | on-chain (`cast`) | |
|-------|-----------|-------------------|---|
| active transcoders | `protocol.num_active_transcoders` = 100 | `BondingManager.getTranscoderPoolSize()` = 100 | ✓ |
| current round | `protocol.current_round` = 4276 | `RoundsManager.currentRound()` = 4276 | ✓ |
| rewardCut / feeShare / activationRound | `transcoder` view (top 3 by reward calls) | `BondingManager.getTranscoder(id)` | ✓ exact |
| deactivationRound (never) | `null` | `2^256-1` sentinel | ✓ equivalent |

Regression checks live in `checks/*.sql` (`nuthatch check`).

## Provenance

Contract set, addresses, start blocks and event lists taken from `livepeer/subgraph`
`networks.yaml` (`arbitrum-one`) and `subgraph.template.yaml`. ABIs vendored from that repo's
`abis/`.
