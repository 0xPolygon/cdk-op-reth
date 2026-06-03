# Prerequisites

Reference material for the deployment: component versions, the wallet accounts you'll need to fund, and the gas token your network will use.

## Component Versions

**OP Stack**

| Component | Version | Docker Image |
|---|---|---|
| op-deployer | v0.6.0 | us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer |
| op-reth | v1.11.5 | us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth |
| op-node | v1.16.11 | us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node |
| op-batcher | v1.16.6 | us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher |

**Polygon Stack**

| Component | Version | Consensus Type | Docker Image |
|---|---|---|---|
| agglayer-contracts | v12.2.3 | PP/FEP | - |
| aggkit | 0.8.2 | PP/FEP | ghcr.io/agglayer/aggkit |
| zkevm-bridge-service | v0.6.3 | PP/FEP | hermeznetwork/zkevm-bridge-service |
| aggkit-prover | v1.9.2 | FEP | ghcr.io/agglayer/aggkit-prover |
| op-succinct-proposer | v3.5.0-agglayer | FEP | ghcr.io/agglayer/op-succinct/op-succinct-agglayer |

## Wallet Accounts

You will need funded wallet accounts to deploy and operate the network.

**Common**

| Role      | Variable Name     | L1 Funds Required | L2 Funds Required |
|-----------|------------------|:-----------------:|:-----------------:|
| Deployer  | DEPLOYER_ADDRESS |        ✅         |        ❌         |

**Polygon Stack**

| Role           | Variable Name         | L1 Funds Required | L2 Funds Required |
|----------------|----------------------|:-----------------:|:-----------------:|
| Aggoracle      | AGGORACLE_ADDRESS    |        ❌         |        ✅         |
| Aggsender      | AGGSENDER_ADDRESS    |        ❌         |        ❌         |
| ClaimTxManager | CLAIMTXMANAGER_ADDR  |        ❌         |        ✅         |
| Admin          | ADMIN_ADDR           |        ❌         |        ❌         |

**OP Stack**

| Role      | Variable Name            | L1 Funds Required | L2 Funds Required |
|-----------|-------------------------|:-----------------:|:-----------------:|
| Batcher   | BATCHER_ADDRESS         |        ✅         |        ❌         |


> 💡 **Tip:** The easiest way to get funds on L2 for testnet/devnet is by prefunding accounts during genesis generation. When you follow step 2 (**[Genesis Generation](02-genesis.md)**), add any accounts you want to be funded directly to the genesis file with the desired balance.

## Gas Token

You must decide which token the network uses as its gas token before deploying. Define the **token address** accordingly:

| Variable Name | Token Address |
|---------------|---------------|
| GAS_TOKEN_ADDRESS | `0x0000000000000000000000000000000000000000` (native ETH) or the ERC-20 contract address on L1 (custom gas token) |

This address is used throughout the setup, so settle on it up front.

## Agglayer Manager

Choose the environment your network attaches to and use the corresponding Agglayer Manager address:

| Environment | Variable Name | Address |
|-------------|---------------|---------|
| Bali (devnet) | AGGLAYER_MANAGER | `0xe2ef6215adc132df6913c8dd16487abf118d1764` |
| Cardona (testnet) | AGGLAYER_MANAGER | `0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff` |
