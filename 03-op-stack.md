# Network Deployment (OP Stack)

> [!WARNING]
> Step 2 of this chapter deploys L1 contracts and consumes real funds from `DEPLOYER_ADDRESS`. The deployment is irreversible on mainnet. Verify your `intent.toml` carefully before proceeding.

This document explains how to deploy an L2 using the OP Stack's `op-deployer` tool. It follows the official Optimism deployment flow and focuses on the steps most relevant to this repository.

## References

- [Optimism L2 Rollup Tutorial](https://docs.optimism.io/operators/chain-operators/tutorials/create-l2-rollup)
- [op-deployer Tool Documentation](https://docs.optimism.io/operators/chain-operators/tools/op-deployer)

> [!NOTE]
> All commands assume you're running from the repository root.

## Overview

1. Initialize an `op-deployer` workdir
2. Edit `intent.toml` with deployment parameters
3. Deploy L1 contracts using `op-deployer apply`
4. Merge existing genesis allocs (e.g., Polygon) into the op-deployer state
5. Generate final `genesis.json` and `rollup.json` files via `op-deployer inspect`

### Environment Variables

Set the following environment variables before proceeding:

```shell
# L1 chain ID — 11155111 for Sepolia, 1 for Ethereum mainnet
export l1_chain_id=<your-l1-chain-id>
export l2_chain_id=<l2ChainID-from-combined.json>
export l1_rpc_url="https://<your_l1_rpc>"
export l1_rpc_url_wss="wss://<your_l1_rpc>"
export deployer_private_key=0x... # private key of DEPLOYER_ADDRESS
# See Component Versions table in 00-prerequisites.md for current values
export op_deployer_version="<op_deployer_version>"
export op_reth_version="<op_reth_version>"
```

## Step 1: Initialize the Deployer Workdir

Create a local `deployer` folder and initialize the op-deployer state:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	init \
		--l1-chain-id ${l1_chain_id} \
		--l2-chain-ids ${l2_chain_id} \
		--workdir /deployer
```

Edit the generated `deployer/intent.toml` with your deployment parameters. Here's an `intent.toml` example file:

```toml
configType = "standard-overrides"
opDeployerVersion = "<generated>"
l1ChainID = <your-l1-chain-id>
opcmAddress = "<generated>"
fundDevAccounts = true
l1ContractsLocator = "embedded"
l2ContractsLocator = "embedded"

[globalDeployOverrides]
  l2BlockTime = 1

[[chains]]
  id = "<generated>"
  baseFeeVaultRecipient = "<ADMIN_ADDRESS>"
  l1FeeVaultRecipient = "<ADMIN_ADDRESS>"
  sequencerFeeVaultRecipient = "<ADMIN_ADDRESS>"
  operatorFeeVaultRecipient = "<ADMIN_ADDRESS>"
  eip1559DenominatorCanyon = 250
  eip1559Denominator = 50
  eip1559Elasticity = 6
  gasLimit = 60000000
  operatorFeeScalar = 0
  operatorFeeConstant = 0
  useRevenueShare = true
  chainFeesRecipient = "<ADMIN_ADDRESS>"
  minBaseFee = 0
  daFootprintGasScalar = 0
  [chains.roles]
    l1ProxyAdminOwner = "<ADMIN_ADDRESS>"
    l2ProxyAdminOwner = "<ADMIN_ADDRESS>"
    systemConfigOwner = "<ADMIN_ADDRESS>"
    unsafeBlockSigner = "<BATCHER_ADDRESS>"
    batcher = "<BATCHER_ADDRESS>"
    proposer = "<ADMIN_ADDRESS>"
    challenger = "<ADMIN_ADDRESS>"
```

## Step 2: Deploy L1 Contracts

Once your `intent.toml` is ready, deploy the L1 contracts required by the OP Stack:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	apply \
		--workdir /deployer \
		--l1-rpc-url ${l1_rpc_url} \
		--private-key ${deployer_private_key}
```

This writes the deployer state to `deployer/state.json`.

## Step 3: Merge OP + Polygon Genesis with Pre-deployed Contracts

This step is required when you have pre-deployed contracts or existing chain allocs (for example, from a `polygon-genesis.json`). You must merge those allocs into the op-deployer state so the final L2 genesis includes the pre-deployed addresses and balances.

The commands below:

1. Extract the base64/gzip-encoded allocs from the op-deployer state
2. Merge them with your Polygon alloc fragment
3. Re-encode and write the merged allocs back into the state

```shell
# Path to the polygon-genesis.json produced in chapter 2
export polygon_genesis_path=<path/to/polygon-genesis.json>

# 1. Extract the existing allocs from the op-deployer state
jq -r '.opChainDeployments[].allocs' deployer/state.json \
  | base64 -d \
  | gzip -d > allocs.json

# 2. Merge with the Polygon genesis and re-encode
jq -s add allocs.json "${polygon_genesis_path}" \
  | gzip \
  | base64 -w 0 > merged-allocs.b64

# 3. Back up the original state and write the merged allocs back
cp deployer/state.json deployer/state.json.bak
jq --rawfile merged merged-allocs.b64 \
   '.opChainDeployments[].allocs = $merged' \
   deployer/state.json > deployer/state.json.new
mv deployer/state.json.new deployer/state.json

# Cleanup
rm allocs.json merged-allocs.b64
```

> [!TIP]
> Verify the merge before continuing: `jq -r '.opChainDeployments[].allocs' deployer/state.json | base64 -d | gzip -d | jq 'keys | length'` should report a higher account count than the same command against `deployer/state.json.bak`. Keep `state.json.bak` until you've confirmed step 4 succeeds.

## Step 4: Generate Final Artifacts

Once the state is ready, use `op-deployer inspect` to produce the final `genesis.json` and `rollup.json` for the L2:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	inspect genesis --workdir /deployer ${l2_chain_id} > ./deployer/genesis.json

docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	inspect rollup --workdir /deployer ${l2_chain_id} > ./deployer/rollup.json
```

These files serve as inputs for running the OP Stack nodes or for further tooling in this repository.

## OP Stack Components

After deploying the contracts and generating the genesis files, you'll need to run the OP Stack components. For complete configuration documentation, refer to the official Optimism documentation:

### Sequencer

> **Source of Truth**: [Spinning up the sequencer](https://docs.optimism.io/chain-operators/guides/deployment/sequencer-node)

The sequencer consists of two core components:
- **op-reth**: Execution layer that processes transactions and maintains state
- **op-node**: Consensus layer that orders transactions and creates L2 blocks

The sequencer is responsible for ordering transactions from users, building L2 blocks, and signing blocks on the P2P network.

### Batcher

> **Source of Truth**: [Spinning up the batcher](https://docs.optimism.io/chain-operators/guides/deployment/spin-batcher)

The batcher (`op-batcher`) collects L2 transactions and submits them as batches to L1. It ensures L2 transaction data is available on L1 for data availability and enables users to reconstruct the L2 state.

---

**Next:** [Rollup Initialization →](04-rollup-initialization.md)
