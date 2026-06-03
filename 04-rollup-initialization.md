# Rollup Initialization

> [!WARNING]
> This step is irreversible. Once the rollup is initialized on-chain, it cannot be undone. Double-check all parameters before proceeding.

This document explains how to generate rollup initialization artifacts and run the rollup initialization script for an Agglayer rollup.

## References

- [Initialize Rollup README](https://github.com/agglayer/agglayer-contracts/blob/main/tools/initializeRollup/README.md) — Full usage details, configuration options, and examples

## Overview

- **PP** deployments: skip to [Step 4](#step-4-prepare-rollup-initialization-json).
- **FEP** deployments: complete the FEP-only setup first ([Steps 1–3](#fep-only-setup)), then continue with Step 4.

---

## FEP-only setup

The next three steps apply only to FEP deployments. PP deployments skip ahead to [Step 4](#step-4-prepare-rollup-initialization-json).

### Step 1: Setup Environment Variables *(FEP only)*

Export the following variables (replace placeholders with values for your environment):

```shell
export l1_rpc_url="https://<your_l1_rpc>"
export l1_beacon_rpc_url="https://<your_l1_beacon_rpc>"
export op_node_url="https://<your_op_node>"
export op_reth_url="http://<your_op_reth>"
# Set to the L2 block at which proof generation should start (typically the L2 genesis block).
# Do not leave at 1 unless you genuinely want to start proving from block 1.
export starting_block_number=<your-starting-block-number>
# See Component Versions table in 00-prerequisites.md for current values
export op_succinct_version="<op_succinct_version>"
```

> [!TIP]
> Keep these values private and do not commit them to version control.

### Step 2: Create Initialization Working Directory *(FEP only)*

Create a directory for the initialization run and write a minimal `.env` file consumed by the op-succinct helper:

```shell
mkdir -p initialize_${op_succinct_version}
cd initialize_${op_succinct_version}

cat > .env <<EOF
L1_RPC="${l1_rpc_url}"
L1_BEACON_RPC="${l1_beacon_rpc_url}"
L2_NODE_RPC="${op_node_url}"
L2_RPC="${op_reth_url}"
STARTING_BLOCK_NUMBER="${starting_block_number}"
EOF
```

### Step 3: Fetch L2 Output-Oracle Configuration *(FEP only)*

Run the op-succinct container to generate/fetch the L2 output-oracle configuration file (`opsuccinctl2ooconfig.json`) into the working directory:

```shell
docker run --rm -it \
    --env OP_SUCCINCT_L2_OUTPUT_ORACLE_CONFIG_PATH=/tmp/env/opsuccinctl2ooconfig.json \
    --platform linux/amd64 \
    -v "$(pwd)":/tmp/env \
    ghcr.io/agglayer/op-succinct/op-succinct-agglayer:${op_succinct_version} \
    /bin/bash -c "fetch-l2oo-config --env-file /tmp/env/.env"
```

After completion, you should have `opsuccinctl2ooconfig.json` (or other output files) in the current directory. This can be used to help construct `aggchainParams.initParams`.

---

## Step 4: Prepare Rollup Initialization JSON

> [!NOTE]
> The commands in this step are expected to be run from the `agglayer-contracts` root directory.

Copy the example initialization JSON file:

```shell
cp ./tools/initializeRollup/initialize_rollup.json.example ./tools/initializeRollup/initialize_rollup.json
```

Edit the file with your values.

**PP example:**

```json
{
    "type": "EOA",
    "trustedSequencerURL": "http://<your_l1_rpc>",
    "networkName": "<network-name>",
    "trustedSequencer": "<AGGSENDER_ADDRESS>",
    "chainID": <l2ChainID-from-combined.json>,
    "rollupAdminAddress": "<ADMIN_ADDRESS>",
    "consensusContractName": "AggchainECDSAMultisig",
    "gasTokenAddress": "0x0000000000000000000000000000000000000000",
    "deployerPvtKey": "",
    "maxFeePerGas": "",
    "maxPriorityFeePerGas": "",
    "multiplierGas": "",
    "timelockDelay": 0,
    "timelockSalt": "",
    "rollupManagerAddress": "<polygonRollupManagerAddress-from-combined.json>",
    "aggchainParams": {
        "useDefaultVkeys": true,
        "useDefaultSigners": false,
        "signers": [
            {
                "addr": "<AGGSENDER_ADDRESS>",
                "url": "https://<your_aggsender_rpc>"
            }
        ],
        "threshold": 1,
        "vKeyManager": "<ADMIN_ADDRESS>"
    }
}
```

**FEP example:**

```json
{
    "type": "EOA",
    "trustedSequencerURL": "http://<your_l1_rpc>",
    "networkName": "<network-name>",
    "trustedSequencer": "<AGGSENDER_ADDRESS>",
    "chainID": <l2ChainID-from-combined.json>,
    "rollupAdminAddress": "<ADMIN_ADDRESS>",
    "consensusContractName": "AggchainFEP",
    "gasTokenAddress": "0x0000000000000000000000000000000000000000",
    "deployerPvtKey": "",
    "maxFeePerGas": "",
    "maxPriorityFeePerGas": "",
    "multiplierGas": "",
    "timelockDelay": 0,
    "timelockSalt": "",
    "rollupManagerAddress": "<polygonRollupManagerAddress-from-combined.json>",
    "aggchainParams": {
        "initParams": {
            "l2BlockTime": <l2BlockTime-from-opsuccinctl2ooconfig.json>,
            "rollupConfigHash": "<rollupConfigHash-from-opsuccinctl2ooconfig.json>",
            "startingOutputRoot": "<startingOutputRoot-from-opsuccinctl2ooconfig.json>",
            "startingBlockNumber": <startingBlockNumber-from-opsuccinctl2ooconfig.json>,
            "startingTimestamp": <startingTimestamp-from-opsuccinctl2ooconfig.json>,
            "submissionInterval": 1,
            "optimisticModeManager": "<ADMIN_ADDRESS>",
            "aggregationVkey": "<aggregationVkey-from-opsuccinctl2ooconfig.json>",
            "rangeVkeyCommitment": "<rangeVkeyCommitment-from-opsuccinctl2ooconfig.json>"
        },
        "useDefaultVkeys": true,
        "useDefaultSigners": false,
        "signers": [
            {
                "addr": "<AGGSENDER_ADDRESS>",
                "url": "https://<your_aggsender_rpc>"
            }
        ],
        "threshold": 1,
        "vKeyManager": "<ADMIN_ADDRESS>"
    }
}
```

## Step 5: Run the Initialization Script

Install dependencies and run the Hardhat initialization script. Substitute `mainnet` for production deployments:

```shell
npm install
npx hardhat run ./tools/initializeRollup/initializeRollup.ts --network sepolia
```

---

**Next:** [OP Succinct Configuration →](05-op-succinct-config.md) *(FEP only)* — PP deployments continue to [Polygon Stack Common Components →](06-polygon-stack-common.md)
