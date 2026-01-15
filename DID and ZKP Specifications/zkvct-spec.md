---
title: ZkVCT Specification
post_type: page
menu_order: 3
post_status: publish
post_excerpt: Technical specification for Zero-Knowledge Verifiable Credential Tokens.
taxonomy:
    category:
        - identity
---

# ZkVCT: Zero-Knowledge Verifiable Credential Token

- [ZkVCT 101: A Primer](#zkvct-101-a-primer)
- [Abstract](#abstract)
- [Architecture & Flow](#architecture--flow)
- [Cryptographic Primitives](#cryptographic-primitives)
- [Trusted Setup (Power of Tau)](#trusted-setup-power-of-tau)
- [Circuit Specifications](#circuit-specifications)
- [On-Chain Verification](#on-chain-verification)
- [Nullifiers & Anonymity](#nullifiers--anonymity)
- [Performance Benchmarks](#performance-benchmarks)
- [Conclusion](#conclusion)

<a id="zkvct-101-a-primer"></a>
## ZkVCT 101: A Primer

*This section provides a high-level introduction to Zero-Knowledge Proofs in the context of ShareRing.*

**The Problem**: To prove you are over 18, you typically have to show your ID card. This reveals your exact date of birth, your name, your address, and your passport number—far more information than the bouncer or website actually needs.

**The Solution**: A **Zero-Knowledge Proof (ZKP)** allows you to prove a statement ("I am over 18") is true *without* revealing the underlying data (your Date of Birth) that makes it true.

**ZkVCT** is ShareRing's implementation of this concept. It allows users to hold a verified credential (like a passport) on their phone and generate cryptographic proofs for third parties. The third party verifies the math, not the document.

<a id="abstract"></a>
## Abstract

**ZkVCT (Zero-Knowledge Verifiable Credential Token)** is a privacy-preserving layer built on top of the ShareLedger identity stack. It leverages **zk-SNARKs** (Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge) to enable **Selective Disclosure** of identity attributes.

Unlike standard Verifiable Credentials (VCs) where the holder presents a signed JSON object containing their data, ZkVCT requires the holder to compute a proof locally. This proof is then submitted to a **Verifier Smart Contract** on ShareLedger (or other supported chains). The contract validates the proof against a known **Merkle Root** or **Credential Hash** without ever accessing the plaintext PII (Personally Identifiable Information).

<a id="architecture--flow"></a>
## Architecture & Flow

The ZkVCT lifecycle involves three key entities: the **Issuer**, the **Holder (Prover)**, and the **Verifier**.

### 1. Issuance (The Commitment)
1.  **Identity Verification**: The User (Holder) undergoes KYC/KYB.
2.  **Merkle Tree Construction**: The Issuer hashes the user's attributes (Name, DOB, Country) into a **Sparse Merkle Tree** (SMT) using the **Poseidon** hash function.
3.  **On-Chain Anchor**: The Issuer signs the **Merkle Root** and anchors it to the user's DID on ShareLedger.
4.  **Credential Delivery**: The raw attributes and the Merkle Branch (sibling hashes) are delivered securely to the Holder's device.

### 2. Proving (Off-Chain)
When a Holder needs to prove a claim (e.g., "Citizenship != 'Blocked Country'"):
1.  **Input Selection**: The Holder selects the private inputs (Citizenship: "Australia") and the public inputs (Merkle Root).
2.  **Circuit Execution**: The ShareRing App runs a **Circom** circuit locally.
3.  **Proof Generation**: Using `snarkjs` (ported to mobile), a `Proof` and `PublicSignals` are generated.

### 3. Verification (On-Chain)
1.  **Submission**: The `Proof` is sent to the ZkVCT Verifier Contract.
2.  **Validation**: The contract checks:
    *   Is the Merkle Root valid and signed by a trusted Issuer?
    *   Does the Proof mathematically hold true for the Public Signals?
    *   Has this Nullifier been used before? (If double-spending protection is active).
3.  **Result**: The contract returns `TRUE`, authorizing the transaction or interaction.

![ZkVCT Architecture Flow](https://sharering.atlassian.net/wiki/download/attachments/3561357317/image-20240813-074144.png?api=v2)

<a id="cryptographic-primitives"></a>
## Cryptographic Primitives

ZkVCT relies on the following standard cryptographic constructions:

*   **Proof System**: **Groth16** (chosen for its small proof size (~128 bytes) and fast verification time).
*   **Elliptic Curve**: **BN254** (also known as BN128). This curve is Ethereum-precompile friendly, allowing cost-effective verification on EVM chains.
*   **Hash Function**: **Poseidon**. A ZK-friendly hash function optimized for arithmetic circuits (R1CS), significantly reducing the constraint count compared to SHA-256.
*   **Merkle Tree**:
    *   **Arity**: 2 (Binary Tree)
    *   **Depth**: 20 (Supporting ~1 million attributes/claims per tree, or used for membership sets).
    *   **Leaf Hash**: `Poseidon([Value, Salt])`

<a id="trusted-setup-power-of-tau"></a>
## Trusted Setup (Power of Tau)

Groth16 requires a **Trusted Setup Ceremony** to generate the Proving Key and Verification Key. If the entropy (toxic waste) from this setup is leaked, a malicious actor could forge false proofs.

### The Ceremony
ShareRing participates in a Multi-Party Computation (MPC) ceremony known as **Power of Tau**.
1.  **Phase 1 (Universal)**: A general setup (e.g., Perpetual Powers of Tau) applicable to all circuits. This establishes the initial randomness.
2.  **Phase 2 (Circuit Specific)**: A setup specific to each ZkVCT circuit (e.g., the "Age Check" circuit).

**Security Guarantee**: As long as *at least one* participant in the ceremony deletes their toxic waste, the system is secure. ShareRing acts as the Coordinator, with community members acting as Contributors.

<a id="circuit-specifications"></a>
## Circuit Specifications

Circuits are written in **Circom 2.0**. Below is a technical breakdown of a standard "Range Check" circuit used for Age Verification.

### Example: Age Verification Circuit

```circom
template AgeCheck() {
    // Private Inputs
    signal input birthDateUnix;
    signal input merklePath[20];
    signal input merkleSiblings[20];
    
    // Public Inputs
    signal input currentDateUnix;
    signal input thresholdAge;
    signal input merkleRoot;

    // 1. Merkle Inclusion Proof
    component leafHasher = Poseidon(1);
    leafHasher.inputs[0] <== birthDateUnix;
    
    component treeVerifier = MerkleTreeVerifier(20);
    treeVerifier.leaf <== leafHasher.out;
    treeVerifier.root <== merkleRoot;
    treeVerifier.pathElements <== merkleSiblings;
    treeVerifier.pathIndices <== merklePath;

    // 2. Age Range Logic
    // 365.25 * 24 * 60 * 60 = 31557600 seconds/year
    signal ageSeconds;
    ageSeconds <== currentDateUnix - birthDateUnix;
    
    signal requiredSeconds;
    requiredSeconds <== thresholdAge * 31557600;

    component greaterThan = GreaterThan(64);
    greaterThan.in[0] <== ageSeconds;
    greaterThan.in[1] <== requiredSeconds;
    
    greaterThan.out === 1;
}
```

**Constraint Count**: ~3,000 R1CS (Poseidon Merkle Proof dominates the cost).

<a id="on-chain-verification"></a>
## On-Chain Verification

The Verifier is a standard CosmWasm (or Solidity) smart contract.

### Contract Interface

```rust
// Rust / CosmWasm Pseudo-code
pub fn verify_proof(
    proof: Groth16Proof,
    public_signals: [u256; N]
) -> StdResult<bool> {
    // 1. Load Verification Key (stored in contract state)
    let vk = load_vk(deps.storage)?;
    
    // 2. Perform Pairing Check
    // e(A, B) == e(alpha, beta) * e(L, gamma) * e(C, delta)
    let valid = pairing_check(vk, proof, public_signals);
    
    // 3. Check Business Logic (e.g. is the Merkle Root trusted?)
    let root = public_signals[0];
    let is_trusted = check_trusted_issuer(root);
    
    Ok(valid && is_trusted)
}
```

### Gas Considerations
*   **ShareLedger**: Native verification is optimized and incurs negligible fees.
*   **Ethereum/EVM**: Verifying a Groth16 proof costs approximately **200k - 300k gas**. This is suitable for high-value transactions (e.g., DeFi compliance) but may require batching for lower-value use cases.

<a id="nullifiers--anonymity"></a>
## Nullifiers & Anonymity

To prevent correlation (tracking a user across multiple verifications) or double-spending, ZkVCT uses **Nullifiers**.

*   **Definition**: A deterministic unique hash generated from the private credential and the specific interaction context.
*   **Formula**: `Nullifier = Poseidon([UserPrivateKey, CredentialID, ContextID])`
*   **Usage**: The Nullifier is revealed as a Public Signal. The Smart Contract records `used_nullifiers[nullifier] = true`. If the same user tries to use the same credential for the same `ContextID` (e.g., claiming an airdrop twice), the contract detects the duplicate nullifier and rejects the proof.
*   **Privacy**: Because the Nullifier relies on the Private Key, no observer can link two different Nullifiers (for different Contexts) to the same user.

<a id="performance-benchmarks"></a>
## Performance Benchmarks

Benchmarks recorded on standard mobile hardware (iPhone 13 / Pixel 6 class):

| Operation | Time |
| :--- | :--- |
| **Proof Generation (Age Check)** | ~1.2 seconds |
| **Proof Generation (Passport Auth)** | ~2.5 seconds |
| **Verification (On-Chain)** | < 100ms (ShareLedger) |
| **Verification (EVM)** | ~300k Gas |

This performance allows for a seamless "Face ID -> Proof -> Login" user experience with minimal latency.

<a id="conclusion"></a>
## Conclusion

ZkVCT represents the convergence of regulatory compliance and user privacy. By utilizing **Groth16 SNARKs** and **Poseidon Hashing**, ShareRing enables a trust model where issuers attest to facts, holders retain control of data, and verifiers receive mathematical guarantees of validity without liability for data storage.

This infrastructure is the backbone of the "ShareRing ID" ecosystem, ensuring that as digital identity scales, it remains secure, private, and sovereign.
