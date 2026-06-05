```
░█▀▀░█▀▄░█░█░░░░░█▀█░█▀█░░░░░█▀▄░█▀▀░▀█▀░█░█
░█░░░█░█░█▀▄░▄▄▄░█░█░█▀▀░▄▄▄░█▀▄░█▀▀░░█░░█▀█
░▀▀▀░▀▀░░▀░▀░░░░░▀▀▀░▀░░░░░░░▀░▀░▀▀▀░░▀░░▀░▀
```

**Deploy an OP Stack rollup, connected to Agglayer.**

This repository provides a step-by-step deployment guide for standing up an OP Stack rollup and attaching it to the [Agglayer](https://agglayer.github.io/). It is intended for chain operators bringing up devnets, testnets, or production networks on **Sepolia** or **Ethereum mainnet**.

> [!IMPORTANT]
> The commands in this guide pass private keys directly on the command line for clarity. For testnet and mainnet deployments, use a keystore or hardware wallet — never literal private keys, and never commit them to version control. See [Prerequisites](00-prerequisites.md) for guidance.

## Architecture

The diagrams below show the high-level architecture of the full setup for each supported consensus type.

### Pessimistic Proof (PP) — AggchainECDSAMultisig

Contract docs: https://agglayer.github.io/protocol-team-docs/smart-contracts/v12/AggchainECDSAMultisig/

PP is the narrow, bridge-safety proof. It doesn't attest to a chain's state transition — it only proves that withdrawal claims a chain makes against the unified bridge are backed by real deposits, so a misbehaving chain can't drain other chains' assets.

```mermaid
%%{init: {'flowchart': {'defaultRenderer': 'elk'}} }%%
flowchart LR
    %% External actors
    User((User))
    L1[L1]
    SPN[Succinct Prover<br/>Network]

    %% Trusted Environment boundary
    subgraph TE["Trusted Environment"]
        direction TB

        subgraph OPSTACK[" "]
            direction LR
            opnode[op-node]
            opreth[op-reth]
            opbatcher[op-batcher]
        end

        subgraph AGGSTACK[" "]
            direction LR
            bridgeservice[bridge-service]
            aggkit[aggkit]
        end

        agglayernode[agglayer-node]
    end

    %% ===== Main flow (solid, numbered) =====
    User -->|"1. Submit txn for sequencing"| opreth
    opnode -->|"2. Sequence blocks"| opreth
    opbatcher -->|"3. Submit sequenced batch"| L1
    aggkit -->|"4. Set GER"| opreth
    aggkit -->|"5. Submit certificate for settlement"| agglayernode
    agglayernode -->|"6. Request Pessimistic proof"| SPN
    agglayernode -->|"7. Submit settlement txn"| L1

    %% ===== Periodical operations (dashed orange) =====
    bridgeservice -.->|"Read bridge events"| L1
    bridgeservice -.->|"Read bridge events"| opreth
    aggkit -.->|"Read GER/bridge events"| L1

    %% ===== Styling =====
    classDef opComp stroke:#e03131,stroke-width:2px,fill:#fff,color:#1e1e1e
    classDef aggComp stroke:#7950f2,stroke-width:2px,fill:#fff,color:#1e1e1e
    classDef l1Box stroke:#1e1e1e,stroke-width:2px,fill:#a5d8ff,color:#1e1e1e
    classDef userCircle stroke:#1e1e1e,stroke-width:2px,fill:#ffd8a8,color:#1e1e1e
    classDef spnBox stroke:#f783ac,stroke-width:2px,fill:#fcc2d7,color:#1e1e1e

    class opnode,opreth,opbatcher opComp
    class aggkit,bridgeservice,agglayernode aggComp
    class L1 l1Box
    class User userCircle
    class SPN spnBox

    style TE stroke:#1e1e1e,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent,color:#1e1e1e
    style OPSTACK stroke:#e03131,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent
    style AGGSTACK stroke:#7950f2,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent

    %% Periodical links rendered in orange dashed (indices 7-9)
    linkStyle 7 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
    linkStyle 8 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
    linkStyle 9 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
```

> [!NOTE]
> The dashed arrows are periodic operations: `bridge-service` and `aggkit` continuously read bridge and GER events from L1 and `op-reth`.

### Full Execution Proof (FEP) — AggchainFEP

Contract docs: https://agglayer.github.io/protocol-team-docs/smart-contracts/v12/AggchainFEP/

FEP is the strong proof: it cryptographically attests to the chain's full state transition using SP1 (Succinct) zero-knowledge proofs, on top of the bridge-safety guarantee the PP provides.

```mermaid
flowchart LR
    %% External actors
    User((User))
    L1[L1]
    SPN[Succinct Prover<br/>Network]

    %% Trusted Environment boundary
    subgraph TE["Trusted Environment"]
        direction TB

        subgraph OPSTACK[" "]
            direction LR
            opnode[op-node]
            opreth[op-reth]
            opbatcher[op-batcher]
        end

        subgraph AGGSTACK[" "]
            direction LR
            bridgeservice[bridge-service]
            aggkit[aggkit]
            aggkitprover[aggkit-prover]
            opsuccinct[op-succinct-<br/>proposer]
        end

        agglayernode[agglayer-node]
    end

    %% ===== Main flow (solid, numbered) =====
    User -->|"1. Submit txn for sequencing"| opreth
    opnode -->|"2. Sequence blocks"| opreth
    opbatcher -->|"3. Submit sequenced batch"| L1
    aggkit -->|"4. Set GER"| opreth
    aggkit -->|"5. Request aggchain proof"| aggkitprover
    aggkitprover -->|"6. Request aggregation Proof"| opsuccinct
    opsuccinct -->|"7. Request aggchain proof"| SPN
    aggkit -->|"8. Submit certificate for settlement"| agglayernode
    agglayernode -->|"9. Request Pessimistic proof"| SPN
    agglayernode -->|"10. Submit settlement txn"| L1

    %% ===== Periodical operations (dashed orange) =====
    bridgeservice -.->|"Read bridge events"| L1
    bridgeservice -.->|"Read bridge events"| opreth
    aggkit -.->|"Read GER/bridge events"| L1
    opsuccinct -.->|"range proof"| SPN

    %% ===== Styling =====
    classDef opComp stroke:#e03131,stroke-width:2px,fill:#fff,color:#1e1e1e
    classDef aggComp stroke:#7950f2,stroke-width:2px,fill:#fff,color:#1e1e1e
    classDef succinctComp stroke:#f783ac,stroke-width:2px,fill:#fff,color:#1e1e1e
    classDef l1Box stroke:#1e1e1e,stroke-width:2px,fill:#a5d8ff,color:#1e1e1e
    classDef userCircle stroke:#1e1e1e,stroke-width:2px,fill:#ffd8a8,color:#1e1e1e
    classDef spnBox stroke:#f783ac,stroke-width:2px,fill:#fcc2d7,color:#1e1e1e

    class opnode,opreth,opbatcher opComp
    class aggkit,aggkitprover,bridgeservice,agglayernode aggComp
    class opsuccinct succinctComp
    class L1 l1Box
    class User userCircle
    class SPN spnBox

    style TE stroke:#1e1e1e,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent,color:#1e1e1e
    style OPSTACK stroke:#e03131,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent
    style AGGSTACK stroke:#7950f2,stroke-width:2px,stroke-dasharray: 8 8,fill:transparent

    %% Periodical links rendered in orange dashed (indices 10-13)
    linkStyle 10 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
    linkStyle 11 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
    linkStyle 12 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
    linkStyle 13 stroke:#f08c00,stroke-width:2px,stroke-dasharray: 4 6
```

> [!NOTE]
> The dashed arrows are returns and periodic operations: the Succinct Prover Network returns the `aggregation`/`range` proofs, and `bridge-service`/`aggkit` continuously read bridge and GER events.

## Choose your path

Pick the consensus type for your rollup and follow the corresponding steps below:

| Consensus | Steps |
|---|---|
| **PP** (AggchainECDSAMultisig) | 1 → 2 → 3 → 4 → 6 |
| **FEP** (AggchainFEP)          | 1 → 2 → 3 → 4 → 5 → 6 → 7 |

## Deployment Steps

Before starting, review the **[Prerequisites](00-prerequisites.md)** for host tooling, component versions, and the wallet accounts you'll need to fund.

1. **[Rollup Creation](01-rollup-creation.md)** — Initiate the Agglayer chain integration process with Polygon Labs to receive the deployment artifacts
2. **[Genesis Generation](02-genesis.md)** — Generate the L2 genesis file with pre-deployed contracts
3. **[OP Stack Deployment](03-op-stack.md)** — Deploy L1 contracts and merge genesis allocs using `op-deployer`
4. **[Rollup Initialization](04-rollup-initialization.md)** — Initialize the rollup on-chain with required parameters
5. **[OP Succinct Configuration](05-op-succinct-config.md)** *(FEP only)* — Add and select the op-succinct configuration for L2 output proposals
6. **[Polygon Stack Common Components](06-polygon-stack-common.md)** — Aggkit configuration for any consensus type
7. **[Polygon Stack FEP Components](07-polygon-stack-fep.md)** *(FEP only)* — Run the Aggkit Prover and OP Succinct Proposer services
