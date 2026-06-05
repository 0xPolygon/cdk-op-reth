# Genesis File Generation

This document describes how to generate the L2 genesis file used by the rollup creation flow. The recommended approach is to merge OP Stack and Polygon genesis files with pre-deployed contracts, ensuring pre-deployed addresses and balances are present in the L2 genesis.

## Overview

- **Recommended**: Merge OP Stack + Polygon genesis with pre-deployed contracts
- **Alternative**: Manual L2 contract deployment (advanced; outside the scope of this guide)

1. Checkout the `agglayer-contracts` repository and install dependencies
2. Create and configure the parameter file using values from `combined.json`
3. Download the allocs file from `cdk-contracts-tooling` (Bali, Cardona, mainnet, etc.) as `genesis-base.json`
4. Run the Hardhat script to generate genesis files
5. Rename the output files to canonical names

## Prerequisites

- Host tooling installed per [Prerequisites](00-prerequisites.md#host-tooling) (`node`, `npm`, `jq`, `gzip`, `base64`, `curl`)
- A `combined.json` produced by the chain integration runbook (see [Rollup Creation](01-rollup-creation.md))

## Step 1: Checkout Repository and Install

Clone the `agglayer-contracts` repository at the version that matches this guide:

```shell
# See Component Versions table in 00-prerequisites.md for current values
export agglayer_contracts_version="<agglayer_contracts_version>"
git clone --depth 1 --branch ${agglayer_contracts_version} https://github.com/agglayer/agglayer-contracts
cd agglayer-contracts
npm install
```

## Step 2: Create Parameter File

Copy the example parameter file:

```shell
cp ./tools/createSovereignGenesis/create-genesis-sovereign-params.json.example ./tools/createSovereignGenesis/create-genesis-sovereign-params.json
```

Update the file with relevant information from your `combined.json`. Example configuration:

```json
{
    "rollupManagerAddress": "<polygonRollupManagerAddress-from-combined.json>",
    "rollupID": <rollupID-from-combined.json>,
    "chainID": <l2ChainID-from-combined.json>,
    "gasTokenAddress": "<gasTokenAddress-from-combined.json>",
    "bridgeManager": "<ADMIN_ADDRESS>",
    "sovereignWETHAddress": "0x0000000000000000000000000000000000000000",
    "sovereignWETHAddressIsNotMintable": false,
    "globalExitRootUpdater": "",
    "globalExitRootRemover": "<ADMIN_ADDRESS>",
    "emergencyBridgePauser": "<ADMIN_ADDRESS>",
    "emergencyBridgeUnpauser": "<ADMIN_ADDRESS>",
    "proxiedTokensManager": "<ADMIN_ADDRESS>",
    "setPreMintAccounts": true,
    "preMintAccounts": [
        {
            "balance": "1000000000000000000",
            "address": "<AGGORACLE_ADDRESS>"
        },
        {
            "balance": "1000000000000000000",
            "address": "<CLAIMTXMANAGER_ADDRESS>"
        }
    ],
    "setTimelockParameters": true,
    "timelockParameters": {
        "adminAddress": "<ADMIN_ADDRESS>",
        "minDelay": 0
    },
    "useAggOracleCommittee": true,
    "aggOracleCommittee": [
        "<AGGORACLE_ADDRESS>"
    ],
    "quorum": 1,
    "aggOracleOwner": "<ADMIN_ADDRESS>",
    "formatGenesis": "geth"
}
```

## Step 3: Download Allocs File

Download the allocs file from the [cdk-contracts-tooling](https://github.com/0xPolygon/cdk-contracts-tooling) repository for your target environment:

- **Bali**: [allocs.json](https://raw.githubusercontent.com/0xPolygon/cdk-contracts-tooling/main/genesis/pp/bali/allocs.json)
- **Cardona**: [allocs.json](https://raw.githubusercontent.com/0xPolygon/cdk-contracts-tooling/main/genesis/pp/cardona/allocs.json)
- **Mainnet**: [allocs.json](https://raw.githubusercontent.com/0xPolygon/cdk-contracts-tooling/main/genesis/pp/mainnet/allocs.json)

Save it as `genesis-base.json` in `./tools/createSovereignGenesis/`:

```shell
# For Bali (example)
curl -o ./tools/createSovereignGenesis/genesis-base.json \
  https://raw.githubusercontent.com/0xPolygon/cdk-contracts-tooling/main/genesis/pp/bali/allocs.json

# For Cardona
# curl -o ./tools/createSovereignGenesis/genesis-base.json \
#   https://raw.githubusercontent.com/0xPolygon/cdk-contracts-tooling/main/genesis/pp/cardona/allocs.json
```

## Step 4: Generate Genesis Files

Run the Hardhat script to assemble the sovereign genesis using your parameter file and the downloaded allocs file (substitute `mainnet` for production deployments):

```shell
npx hardhat run ./tools/createSovereignGenesis/create-sovereign-genesis.ts --network sepolia
```

The script produces `genesis-rollupID-*.json` and `output-rollupID-*.json` files under `./tools/createSovereignGenesis/` (the tool appends your `rollupID` and a datetime to each name).

## Step 5: Rename Outputs

Rename the generated artifacts to the canonical names used by other scripts in this repository:

```shell
mv ./tools/createSovereignGenesis/genesis-rollupID-*.json ./tools/createSovereignGenesis/polygon-genesis.json
mv ./tools/createSovereignGenesis/output-rollupID-*.json ./tools/createSovereignGenesis/polygon-genesis-info.json
```

---

**Next:** [OP Stack Deployment →](03-op-stack.md)
