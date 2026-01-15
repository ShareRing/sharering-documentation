---
title: ShareRing Technical Overview
post_type: page
menu_order: 1
post_status: publish
post_excerpt: A high-level technical overview of the ShareRing ecosystem, including ShareLedger, ID infrastructure, and the App.
taxonomy:
    category:
        - technical
---

# ShareRing Technical Overview

The purpose of this document is to provide a comprehensive, deep-dive technical overview of ShareRing's Identity Service Provider (IdSP) solution. It is designed for both business leaders seeking architectural understanding and technical stakeholders (developers, auditors, node operators) requiring implementation details.

- [Introduction](#introduction)
- [System Architecture](#system-architecture)
- [Global Hybrid Infrastructure](#global-hybrid-infrastructure)
- [ShareLedger Core](#shareledger-core)
- [Identity & Privacy Engine](#identity-verification-engine)
- [Cross-Chain Interoperability](#cross-chain-interoperability)
- [Platform Components](#platform-components)
- [Further Reading](#further-reading)

<a id="introduction"></a>
## Introduction

ShareRing is a digital identity ecosystem built on **ShareLedger**, an application-specific blockchain powered by the [**Tendermint BFT** consensus engine](https://docs.tendermint.com/) and [**Cosmos SDK**](https://docs.cosmos.network/). The platform is designed to solve the "Self-Sovereign Identity" (SSI) challenge: proving who you are without exposing sensitive data.

The system relies on a **Zero-Knowledge (ZK)** architecture where personal data resides exclusively on the user's device (User Sovereignty). The blockchain stores only cryptographic proofs and anchors, ensuring that verification is immutable and tamper-proof while remaining GDPR/PDPA compliant.

<a id="system-architecture"></a>
## System Architecture

The ShareRing platform is a multi-layered stack designed for scalability, security, and interoperability.

1.  **Trust Layer (ShareLedger)**: The immutable ledger for identity anchors, governance, and value exchange.
2.  **Logic Layer (Smart Contracts)**: CosmWASM contracts handling complex business logic (NFTs, SBTs).
3.  **Bridge Layer**: Relayers and IBC protocols connecting ShareLedger to external networks (Ethereum, Cronos, Hedera).
4.  **Application Layer**: The ShareRing Pro App, Web3 Wallets, and third-party integrations via SDKs.

![ShareRing System Architecture](https://sharering.atlassian.net/wiki/download/attachments/4091412482/image-20250924-102248.png?api=v2)

<a id="global-hybrid-infrastructure"></a>
## Global Hybrid Infrastructure

To ensure high availability (99.99%), censorship resistance, and disaster recovery, ShareRing employs a **Multi-Cloud Hybrid Strategy**. This infrastructure mitigates the risk of single-provider failure (e.g., if AWS goes down, the network persists).

### 1. Cloud Provider Diversification
The network is distributed across major cloud providers to ensure decentralization at the infrastructure level:

*   **Google Cloud Platform (GCP)**: Hosts the application backend, API gateways, and CI/CD pipelines.
    *   **Region Strategy**: Services are replicated across multiple zones (e.g., `asia-northeast`, `europe-west`) to reduce latency for global users.
    *   **Orchestration**: All microservices are containerized and managed via **Google Kubernetes Engine (GKE)** with auto-scaling policies triggered by CPU/Memory load.
*   **Multi-Provider Blockchain Infrastructure**: ShareLedger blockchain nodes are distributed across multiple cloud providers, including **Tencent Cloud**, **Google Cloud Platform (GCP)**, **Vultr**, and others.
    *   This distributed approach provides two key benefits: First, the blockchain remains accessible even if the application layer experiences outages, ensuring continuous blockchain-specific operation. Second, if one cloud provider experiences issues, the blockchain continues to operate through other providers, providing built-in redundancy and resilience.

### 2. Network Security Architecture
The infrastructure is segmented into strict security zones:

*   **Public Subnet**:
    *   **Load Balancers (L7)**: Terminate SSL/TLS and route traffic.
    *   **Sentry Nodes**: Act as a shield for Validator nodes. They are the only nodes exposed to the public internet. They mitigate DDoS attacks by filtering traffic before it reaches the consensus layer.
*   **Private Subnet**:
    *   **Validator Nodes**: The core consensus participants. These nodes communicate *only* with Sentry Nodes via private VPN/VPC peering. They hold the signing keys (often secured by **HSMs** - Hardware Security Modules) to propose and sign blocks.
    *   **Application Logic**: Authentication services, Wallet services, and Business logic microservices.
*   **Data Subnet**:
    *   **MongoDB**: Stores non-personally identifiable information (non-PII) required for business logic. The only personally identifiable data stored includes user email addresses, phone numbers, and corresponding ShareLedger public keys—necessary for user sign-up and authentication.
    *   **Redis**: Provides high-performance caching for frequently accessed data. All databases are strictly firewalled with no external access.

### 3. Bridge Architecture (The "Observer" Pattern)
Bridging the centralized backend (Web2) with the decentralized blockchain (Web3) requires robust middleware:
*   **Blockchain Observers**: Specialized services that scan every block on ShareLedger. They listen for specific events (e.g., "Identity Created", "Payment Received") and update the off-chain state in MongoDB.
*   **Relayers**: Bidirectional agents that monitor cross-chain bridges. For example. if a user moves assets from ShareLedger to Cronos, Ethereum, BSC or another chain, the Relayer submits the proof to the destination chain.

![Network Infrastructure](/_images/ShareRing-Infra-202505.png "Network Infrastructure").

<a id="shareledger-core"></a>
## ShareLedger Core

ShareLedger is a sovereign blockchain built on the **Cosmos SDK** framework, which uses Tendermint as its consensus engine. This architecture provides superior performance (10,000+ TPS) and instant finality compared to Proof-of-Work blockchains.

### 1. Consensus Mechanism (Tendermint BFT)
ShareLedger uses **Tendermint Core**, a Practical Byzantine Fault Tolerant (PBFT) consensus engine.
*   **Instant Finality**: unlike Bitcoin (Probabilistic Finality), once a block is committed on ShareLedger, it cannot be reverted. This is critical for Identity and Financial use cases.
*   **Validator Set**: A set of 50+ validators participate in consensus. To create a block, >2/3 of the voting power (weighted by staked SHR) must sign the block.

### 2. Cosmos Modules

#### 2.1 Built-in Modules
ShareLedger utilizes standard Cosmos SDK modules that provide core blockchain functionality:

*   **Auth**: Handles account authentication and transaction signature verification.
*   **Authz**: Enables accounts to authorize other accounts to perform actions on their behalf.
*   **Bank**: Manages token transfers and balances.
*   **Capability**: Implements object capability patterns for secure access control.
*   **Crisis**: Provides emergency mechanisms to halt the blockchain if critical invariants are violated.
*   **Distribution**: Manages the distribution of transaction fees and staking rewards to validators and delegators.
*   **Epoching**: Allows modules to schedule messages for execution at specific block heights.
*   **Evidence**: Handles detection and reporting of validator misbehavior, such as double signing.
*   **Feegrant**: Allows accounts to grant fee allowances to other accounts, enabling sponsored transactions.
*   **Governance**: Enables on-chain proposals and voting for protocol upgrades and parameter changes.
*   **Mint**: Controls the creation of new staking tokens according to the inflation schedule.
*   **Params**: Provides a global parameter store for configurable protocol parameters.
*   **Slashing**: Implements punishment mechanisms for validators who misbehave or go offline.
*   **Staking**: Implements the Proof-of-Stake consensus layer, including validator bonding and delegation.
*   **Upgrade**: Handles coordination of software upgrades across the network.
*   **IBC**: Implements the Inter-Blockchain Communication protocol, enabling secure data and token transfers between Cosmos-based chains.

#### 2.2 ShareLedger Custom Modules
ShareLedger includes custom modules designed specifically for its identity and token management needs:

*   **Electoral**: Manages roles and permissions within the ShareLedger network.
*   **Gentlemint**: Handles the management of ShareLedger's native token (SHR) and transaction fee structures.
*   **Swap**: Enables token swaps between ShareLedger and external chains like Ethereum and Binance Smart Chain (BSC).
*   **DistributionX**: Distributes rewards from smart contract transactions to relevant parties.
*   **Fee**: Provides configurable global transaction fees that can be adjusted across all network nodes.
*   **CosmWasm**: Enables smart contract execution using the CosmWasm virtual machine, allowing developers to build decentralized applications on ShareLedger.

### 3. Smart Contracts (CosmWASM)
ShareLedger supports **CosmWASM**, a WebAssembly-based smart contract engine.
*   **Language**: Contracts are written in **Rust**, offering memory safety and performance superior to Solidity.
*   **Use Cases**:
    *   **Soulbound Tokens (SBTs)**: Non-transferable tokens representing credentials (e.g., "Verified over 18", "Accredited Investor").
    *   **Access Control**: Contracts that gate content or physical access based on VCT (Verifiable Credential Token) ownership.
    *   **Reusable KYC**: A single onboarding process that allows a user to mint a credential used across multiple DeFi platforms or exchanges without re-submitting documents.
    *   **Event Ticketing**: NFT tickets that are cryptographically bound to a user's verified identity, preventing scalping and fraud.

<a id="identity-verification-engine"></a>
## Identity & Privacy Engine

The core value of ShareRing is the ability to verify identity without revealing data. This is achieved through a combination of AI Verification and Zero-Knowledge Cryptography.

### 1. Verification Workflow (The "Trust Anchor")
Before a digital identity is created, the raw data must be verified.
1.  **OCR & Data Extraction**: Neural networks extract text from 700+ ID types (Passports, National IDs).
2.  **Liveness Detection (Passive & Active)**:
    *   **3D Face Map**: The app scans the user's face to ensure depth (prevents "mask" attacks).
    *   **Challenge-Response**: The user must perform a randomized action (e.g., "Turn head left").
3.  **Facial Similarity**: A 1:1 biometric comparison between the Selfie and the ID Photo using deep learning models (ResNet architecture).

### 2. Zero-Knowledge Proofs (ZK-SNARKs)

ShareRing employs Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge (ZK-SNARKs) to enable privacy-preserving verification of identity attributes without disclosing underlying personal data. Based on the referenced document:

* **zkVCT Model Integration**
  ZK-SNARKs are integrated into the Verifiable Credential Token (VCT) model, resulting in Zero-Knowledge Verifiable Credential Tokens (zkVCTs). These tokens support selective disclosure of claims such as age, residency, or accreditation.

* **Trusted Setup (Power of Tau)**
  The system relies on a Power of Tau trusted setup ceremony composed of two phases.

  * A minimum of two contributors and one coordinator are required.
  * Contributors may include the end user and ShareRing; the coordinator may be a third party or ShareRing during early stages.
  * Each contribution injects fresh randomness, preserving security as long as at least one contributor is honest.

* **Statement-Specific Proofs**
  Proofs are generated per statement, for example “age ≥ 18.”
  Each statement results in distinct proving keys and proofs, reducing correlation and replay risks.

* **On-Chain Verification**
  Verification keys are deployed on-chain through dedicated verifier smart contracts.
  Relying parties validate zk-proofs against these contracts without accessing underlying personal data.

* **Privacy Guarantees**
  Only boolean or constraint-satisfaction outcomes are revealed.
  No raw identity attributes or personally identifiable information are stored on-chain.

* **Implementation Status**
  The ZK-SNARK and zkVCT components are currently under active development.

### 3. ShareRing DID Method (`did:shr`)
ShareRing implements the W3C Decentralized Identifier standard.
*   **Format**: `did:shr:<unique-identifier>`
*   **Resolution**: The DID Document is stored on ShareLedger. It contains the **Public Keys** and **Service Endpoints**, allowing anyone to verify signatures made by that identity.

<a id="cross-chain-interoperability"></a>
## Cross-Chain Interoperability

ShareLedger is not an island. It acts as an Identity Hub for the broader Web3 ecosystem.

### 1. The Relayer Model (EVM & Hedera)
Since many target networks (Ethereum, Cronos EVM, Hedera) are not natively IBC-compatible, ShareRing uses a custom **Relayer Bridge**. EVM-compatible contracts deployed on Hedera act as mirror storage for ShareLedger identity data. VCTs and dVCTs minted on ShareLedger are correspondingly minted on Hedera, allowing Hedera-based applications to access credential state locally. ShareLedger remains the authoritative source, while Hedera maintains a synchronized representation for execution efficiency.

### 2. Cronos Integration
ShareRing maintains deep integration with the Cronos ecosystem, connecting to Cronos zkEVM through a relayer or bridge today, with a planned upgrade to Hyperbridge’s ISMP protocol for trustless L2 communication.

<a id="platform-components"></a>
## Platform Components

### ShareRing App (The "Vault")
The user's mobile device acts as a hardware wallet and data vault.
*   **Hardware-Backed Secure Storage**: Seedphrases, private keys, and biometric data are stored using the device's native secure storage: **Keychain** on iOS and **SecureStore** on Android. These systems leverage hardware-backed security: the **Secure Enclave** on iOS and **TEE (Trusted Execution Environment)** on Android which provides the highest level of security for sensitive data storage on mobile devices with minimal risk of compromise.
*   **Local Storage**: ID documents are AES-256 encrypted and never leave the device.

### ShareRing LINK
**ShareRing LINK** is a workflow-based system that enables **Relying Parties (RPs)** to request and receive verified credential data from end-users with their consent. ShareRing LINK acts as a **bridge** that orchestrates the credential request process and delivers results to the RP through multiple delivery methods.

#### Workflow Structure
Workflows are composed of multiple **tasks**, each with defined input and output schemas that specify how data is requested and how results are structured:
*   **Input Schema**: Defines what data or actions are required from the user for each task
*   **Output Schema**: Defines the structure of data produced by each task
*   **Task Types**: Include verification tasks (e.g., age verification, identity document capture), control flow tasks (e.g., conditional branching), and delivery tasks (e.g., webhook, email with downloadable JSON)

#### Delivery Methods
ShareRing LINK supports multiple delivery methods to transmit workflow results to the Relying Party:
*   **Webhook Delivery**: Sends workflow results via HTTP POST to a configured webhook URL with optional OAuth authentication and HMAC signature verification
*   **Email with Downloadable JSON**: Emails a downloadable JSON file containing the workflow output to a specified recipient
*   **And more to come**

#### Example Workflow: Age Verification

A simple workflow for age verification with webhook delivery demonstrates the complete flow from user input to RP receipt:

**Workflow Definition (JSON)**:
```json
{
  "name": "Age Verification Workflow",
  "nodes": [
    {
      "id": "start-1",
      "type": "start",
      "data": {
        "label": "Start",
        "config": { "requiredInputs": [] }
      }
    },
    {
      "id": "age_verification-1",
      "type": "age_verification",
      "data": {
        "label": "Verify Age (18+)",
        "config": {
          "valueType": "age",
          "operator": "gte",
          "compareValue": 18,
          "label": "Please enter your date of birth"
        }
      }
    },
    {
      "id": "delivery_webhook-1",
      "type": "delivery_webhook",
      "data": {
        "label": "Send to RP",
        "config": {
          "url": "https://api.hotel.com/verify",
          "payloadSchema": {
            "type": "object",
            "properties": {
              "age_verified": { "type": "boolean" },
              "verifiedAt": { "type": "string", "format": "date-time" }
            }
          }
        },
        "bindings": [
          {
            "target": "input.payload.age_verified",
            "source": { "ref": "taskOutput", "nodeId": "age_verification-1", "path": "value" }
          },
          {
            "target": "input.payload.verifiedAt",
            "source": { "ref": "taskOutput", "nodeId": "age_verification-1", "path": "verifiedAt" }
          }
        ]
      }
    },
    {
      "id": "end_ok-1",
      "type": "end_ok",
      "data": {
        "label": "End",
        "config": {
          "publicMessage": "Verification complete"
        }
      }
    }
  ],
  "edges": [
    { "source": "start-1", "target": "age_verification-1" },
    { "source": "age_verification-1", "target": "delivery_webhook-1" },
    { "source": "delivery_webhook-1", "target": "end_ok-1" }
  ]
}
```

**End-User Input Flow**:

1. **Workflow Start**: When a user scans a QR code or opens a ShareRing LINK, the workflow run begins. The system identifies the next task requiring user input.

2. **Client Action Request**: The end-user's app receives a client action specifying what data is needed:
    ```json
    {
      "nodeId": "age_verification-1",
      "taskType": "age_verification",
      "label": "Verify Age (18+)",
      "config": {
        "valueType": "age",
        "operator": "gte",
        "compareValue": 18,
        "label": "Please enter your date of birth"
      },
      "input": {},
      "inputSchema": {
        "type": "object",
        "properties": {
          "dob": { "type": "string", "format": "date" }
        },
        "required": ["dob"]
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "value": { "type": "boolean" },
          "verifiedAt": { "type": "string", "format": "date-time" }
        },
        "required": ["value", "verifiedAt"]
      }
    }
    ```

3. **User Provides Data**: With user consent (requiring biometric authentication or PIN), the app extracts the date of birth from the user's stored credentials, calculates the user's age, and executes the verification logic (checking if age >= 18). The app then displays a consent screen asking the user whether they agree to share the verification result. The user can approve or deny sending the exact boolean value (`true` for "yes" or `false` for "no").

4. **Task Completion**: The app submits the task output:
    ```json
    {
      "output": {
        "value": true,
        "verifiedAt": "2024-12-17T10:00:00Z"
      }
    }
    ```
    The workflow engine processes this output, verifies the age is >= 18, and proceeds to the webhook delivery task.

**Relying Party Receipt**:

5. **Webhook Delivery**: The workflow engine constructs the payload using bindings that map the age verification output to the webhook payload structure, then sends an HTTP POST request to the RP's webhook endpoint:

    **Request to RP's Webhook** (`https://api.hotel.com/verify`):
    ```json
    {
      "age_verified": true,
      "verifiedAt": "2024-12-17T10:00:00Z"
    }
    ```

    The webhook may include optional security headers:
    - `x-workflow-signature`: HMAC-SHA256 signature for request verification
    - `x-workflow-timestamp`: Timestamp used in signature generation
    - `Authorization`: Bearer token (if OAuth is configured)

6. **RP Processing**: The Relying Party receives the webhook, verifies the signature (if enabled), and processes the verification result to grant or deny access based on the `age_verified` boolean value.

The workflow structure defines how each task connects, what data flows between tasks via bindings, and how the final result is delivered to the Relying Party through the configured delivery method.

<a id="further-reading"></a>
## Further Reading & Documentation

This technical overview is a living document, evolving alongside the ShareRing ecosystem. We are continuously releasing detailed specifications and guides to support our developer community and partners.

For deeper technical dives, please explore the following resources within our documentation repository:
*   **[Zero Knowledge Verifiable Tokens (zkVCT)](../identity/zkvct-spec.md)**: A detailed look at the cryptographic primitives ensuring user privacy.
*   **[ShareRing DID Method Specification](../identity/did-method-spec.md)**: The formal specification for `did:shr`.
*   **[ShareLedger & Interoperability](../shareledger-and-cronos-interoperability.md)**: Specifics on bridging and cross-chain communication.

We encourage you to check back regularly for updates on Smart Contract templates, SDK documentation, and new architectural modules.
