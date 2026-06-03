# Rollup Network Creation

This document explains how to attach a new rollup to the Agglayer and how to capture the on-chain values produced by that transaction into a `combined.json` file used throughout the rest of this runbook.

## References

- [Initiate Chain Integration with Agglayer](https://github.com/agglayer/runbooks/blob/main/operations/initiate-chain-integration-agglayer.md) — runbook for creating the rollup on L1

## TL;DR

1. Follow the Agglayer runbook to create the rollup on L1 and note the resulting transaction hash
2. Fetch the on-chain values from the creation transaction and write them to `combined.json`

## Step 1: Create the Rollup on L1

Follow the Agglayer runbook to initiate the attachment of a new rollup:

<https://github.com/agglayer/runbooks/blob/main/operations/initiate-chain-integration-agglayer.md#part-1-chain-integration-process>

When the rollup is created, the agglayerManager emits a `CreateNewAggchain` event. Keep the **transaction hash** of the creation transaction — you'll use it to fetch the on-chain values below.

## Step 2: Fetch on-chain values

Export the following variables (replace placeholders with your values):

```shell
export l1_rpc_url="https://<your_l1_rpc>"          # L1 RPC endpoint
export txn="0x..."                                 # Rollup creation transaction hash
export agglayer_manager="<AGGLAYER_MANAGER>"
# Event signature hash (topic0) of the CreateNewAggchain event
export topic0=$(cast sig-event "CreateNewAggchain(uint32,uint32,address,uint64,uint8,bytes)")
```

Fetch the transaction receipt and select the `CreateNewAggchain` log by its `topic0`:

```shell
log=$(
  cast receipt "$txn" \
    --rpc-url "$l1_rpc_url" \
    --json \
  | jq --arg topic0 "$topic0" '
      .logs[]
      | select(.topics[0] == $topic0)
    '
)

# Inspect the matched log (optional)
echo "$log" | jq .
```

Extract the indexed `rollupID` from `topics[1]` and decode the remaining fields from `data`:

```shell
rollup_id=$(jq -r '.topics[1]' <<< "$log" | xargs cast to-dec)
rollup_data=$(jq -r '.data' <<< "$log")
rollup_decoded_data=$(cast abi-decode 'f()(uint32,address,uint64,uint8,bytes)' "$rollup_data" --json)
```

Assemble the values into `combined.json`. The keys below match the names referenced by the rest of this runbook:

```shell
jq -n \
  --argjson r "$rollup_decoded_data" \
  --arg rollup_id "$rollup_id" \
  --arg agglayer_manager "$agglayer_manager" \
  '{
    rollupID: ($rollup_id | tonumber),
    rollupAddress: $r[1],
    l2ChainID: $r[2],
  }' > combined.json
```

Verify the result:

```shell
cat combined.json | jq .
```

Example output:

```json
{
  "rollupID": 77,
  "rollupAddress": "0x9eFbcC7186c08a450E4fFc46DFE89960B50c4A3D",
  "l2ChainID": 777777
}
```

> **Note**: Keep `combined.json` handy — subsequent steps (genesis generation, OP Stack deployment, rollup initialization) read `rollupID`, `l2ChainID` and `rollupAddress` from it.
