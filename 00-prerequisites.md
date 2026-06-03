# Prerequisites

Reference material for the deployment: supported L1s, host tooling, component versions, secrets handling, and the wallet accounts you'll need to fund.

## Supported L1s

This guide targets the following L1 networks:

| L1 | Chain ID |
|---|---|
| Sepolia | `11155111` |
| Ethereum mainnet | `1` |

Examples in subsequent chapters use Sepolia by default. Substitute the mainnet chain ID and an RPC endpoint when deploying to mainnet.

## Host Tooling

Install the following on the machine you'll run the deployment from:

| Tool | Purpose | Notes |
|---|---|---|
| `docker` | Run op-deployer, op-succinct, and component containers | Required |
| `node` + `npm` | Run the Hardhat scripts in `agglayer-contracts` | Node.js ≥ 20 (LTS) recommended |
| `foundry` (`cast`) | On-chain calls in the op-succinct config step | Required for chapter 5 |
| `jq` | Manipulate genesis JSON / op-deployer state | Required |
| `gzip` + `base64` | Decode/re-encode op-deployer allocs | Required |
| `curl` | Download allocs files | Required |
| `git` | Clone `agglayer-contracts` | Required |

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

## Secrets Handling

This guide shows commands that pass `--private-key 0x...` directly on the command line. That's fine for an isolated devnet, but for testnet and mainnet:

- Use a **keystore** (`cast wallet`) or a **hardware wallet** instead of literal private keys.
- Never paste a real private key into a shared terminal, screenshot, or chat.
- Never commit `.env` files containing private keys; add them to `.gitignore`.
- Treat the `DEPLOYER`, `ADMIN`, and Aggchain manager keys as the most sensitive — compromise of any of them is unrecoverable.

## Wallet Accounts

You will need funded wallet accounts to deploy and operate the network.

**Common**

| Role      | Variable Name      | L1 Funds Required | L2 Funds Required |
|-----------|--------------------|:-----------------:|:-----------------:|
| Deployer  | `DEPLOYER_ADDRESS` |        ✅         |        ❌         |

**Polygon Stack**

| Role           | Variable Name             | L1 Funds Required | L2 Funds Required |
|----------------|---------------------------|:-----------------:|:-----------------:|
| Aggoracle      | `AGGORACLE_ADDRESS`       |        ❌         |        ✅         |
| Aggsender      | `AGGSENDER_ADDRESS`       |        ❌         |        ❌         |
| ClaimTxManager | `CLAIMTXMANAGER_ADDRESS`  |        ❌         |        ✅         |
| Admin          | `ADMIN_ADDRESS`           |        ❌         |        ❌         |

**OP Stack**

| Role      | Variable Name      | L1 Funds Required | L2 Funds Required |
|-----------|--------------------|:-----------------:|:-----------------:|
| Batcher   | `BATCHER_ADDRESS`  |        ✅         |        ❌         |

> [!TIP]
> The easiest way to get funds on L2 for devnets and testnets is by prefunding accounts during genesis generation. When you follow step 2 (**[Genesis Generation](02-genesis.md)**), add any accounts you want to be funded directly to the genesis file with the desired balance.

---

**Next:** [Rollup Creation →](01-rollup-creation.md)
