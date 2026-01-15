---
title: CW1056 Identity Standard
post_type: page
menu_order: 2
post_status: publish
post_excerpt: Specification for the CosmWasm smart contract managing ShareRing identities.
taxonomy:
    category:
        - identity
---

# CW1056: ShareRing Lightweight Blockchain Identity Registry

This document defines the standard for the ShareRing Lightweight Blockchain Identity Registry (CW1056), a robust system for managing keys and attributes specifically designed for the ShareRing Network.

## Table of Contents

1. [Simple Summary](#simple-summary)
2. [Abstract](#abstract)
3. [Motivation](#motivation)
4. [Definitions](#definitions)
5. [Specification](#specification)
   - [Messages](#messages)
   - [Queries](#queries)
6. [Events](#events)
   - [did_owner_changed](#did_owner_changed)
   - [did_delegate_changed](#did_delegate_changed)
   - [did_attribute_changed](#did_attribute_changed)
7. [DID Document Generation](#did-document-generation)
   - [Efficient Event Lookup](#efficient-event-lookup)
   - [Building the Document](#building-the-document)
8. [Rationale](#rationale)

<a id="simple-summary"></a>
## Simple Summary

A registry for managing keys and attributes of lightweight blockchain identities on the ShareRing Network. It provides a gas-efficient method for handling infinite delegation and attribute storage without bloating on-chain account abstraction.

<a id="abstract"></a>
## Abstract

Derived from Ethereum’s **ERC1056**, this standard has been modified to align with CosmWasm’s Rust-based smart contracts and ShareRing’s unique requirements. It establishes a protocol for creating and updating identities while minimizing the consumption of blockchain resources. Each identity can support an unlimited number of delegates and associated attributes.

Identity creation is as simple as generating a standard Shareledger key pair. Consequently, identity creation is essentially free (incurring no gas costs), and all Shareledger accounts are inherently valid identities. Furthermore, CW1056 is fully DID-compliant.

Unlike the ERC1056 standard, which utilizes Ethereum addresses as identifiers, this standard uses the public key. Because addresses in the Cosmos ecosystem vary based on chain-specific prefixes, utilizing the public key ensures seamless cross-chain compatibility and interoperability.

<a id="motivation"></a>
## Motivation

The ShareRing Lightweight Blockchain Identity Registry addresses the need for a cost-effective, flexible, and cross-chain compatible solution for identity and attribute management within the Cosmos ecosystem. Traditional smart contract wallets (like Gnosis Safe) can be expensive to deploy for every user. CW1056 solves this by using a single registry contract for all users.

Additionally, this standard supports robust key rotation, allowing users to recover their accounts if they lose their device. By swapping the "Controller" key without changing the underlying Identity ID, users maintain continuity.

<a id="definitions"></a>
## Definitions

-   **Identity**: A unique identifier for an entity, represented by a public key.
    
-   **Delegate**: An address authorized for a specific duration to perform functions on behalf of an identity.
    
-   **Delegate Type**: The classification of a delegate (e.g., `did-jwt`, `raiden`), typically defined by higher-level protocols or applications.
    
-   **Attribute**: A specific piece of data or metadata associated with an identity.
    
-   **Signature Data**: Cryptographic proof that follows the **EIP-191** standard but omits the `0x19` prefix.

<a id="specification"></a>
## Specification

This standard specifies a singleton contract named `ShareledgerDIDRegistry`. Once deployed, it serves as a universal registry accessible to all users on the network.

<a id="messages"></a>
### Messages

**Function**

**Parameters**

**Description**

`ChangeOwner`

`{identity, new_owner}`

Transfers ownership of the specified identity to a new Shareledger account.

`ChangeOwnerSigned`

`{identity, new_owner, signature_data}`

Transfers ownership using a raw signature for authentication.

`AddDelegate`

`{identity, delegate_data}`

Registers a new delegate. `delegate_data` includes `{delegate_type, delegate, validity}` (validity in seconds).

`AddDelegateSigned`

`{identity, delegate_data, signature_data}`

Adds a delegate using a raw signature for authentication.

`RevokeDelegate`

`{identity, delegate_type, delegate}`

Revokes the authorization of a specific delegate for an identity.

`RevokeDelegateSigned`

`{identity, delegate_type, delegate, signature_data}`

Revokes a delegate using a raw signature for authentication.

`SetAttribute`

`{identity, attribute_data}`

Assigns an attribute. `attribute_data` includes `{name, value, validity}`.

`SetAttributeSigned`

`{signature_data}`

Sets an attribute using a raw signature for authentication.

`RevokeAttribute`

`{identity, name, value}`

Removes a previously set attribute.

`RevokeAttributeSigned`

`{identity, name, value, signature_data}`

Revokes an attribute using a raw signature for authentication.

<a id="queries"></a>
### Queries

-   **`IdentityOwner{identity}`**: Retrieves the current owner of the specified identity.
    
-   **`ValidDelegate{identity, delegate_type, delegate}`**: Returns a boolean indicating if a specific delegate is currently authorized under the given type for that identity.

<a id="events"></a>
## Events

<a id="did_owner_changed"></a>
### `did_owner_changed`

**MUST** be triggered upon a successful call to `change_owner` or `change_owner_signed`.

-   **identity**: The public key identifying the identity.
    
-   **owner**: The Shareledger address of the new owner.
    
-   **previous_change**: The block height at which the previous ownership change occurred.

<a id="did_delegate_changed"></a>
### `did_delegate_changed`

**MUST** be triggered whenever a delegate is added or revoked.

-   **identity**: The public key identifying the identity.
    
-   **delegate**: The Shareledger address granted or revoked permission.
    
-   **valid_to**: The block number at which the delegation expires.
    
-   **previous_change**: The block height of the previous delegate modification.

<a id="did_attribute_changed"></a>
### `did_attribute_changed`

**MUST** be triggered whenever an attribute is set or revoked.

-   **identity**: The public key identifying the identity.
    
-   **name**: The key/name of the attribute.
    
-   **value**: The value assigned to the attribute.
    
-   **valid_to**: The block number at which the attribute expires.
    
-   **previous_change**: The block height of the previous attribute modification.

<a id="did-document-generation"></a>
## DID Document Generation

<a id="efficient-event-lookup"></a>
### Efficient Event Lookup

The construction of a DID document is streamlined by tracking historical events:

1.  **Primary Owner Key**: Retrieve the primary owner key via the `identity_owner(identity)` query. This key **MUST** be listed as the first public key in the DID document.
    
2.  **Additional Keys**: Iterate through `did_delegate_changed` events to compile secondary keys and authentication sections. (Specific delegate types for inclusion are pending finalization).
    
3.  **Service Entries & Attributes**: Iterate through `did_attribute_changed` events to populate service endpoints, encryption keys, and other public metadata. (Specific attribute names for inclusion are pending finalization).

<a id="building-the-document"></a>
### Building the Document

The resolver should first look up the primary owner key via `identity_owner(identity)`. Following the primary key, the resolver should process `did_delegate_changed` and `did_attribute_changed` events to build a comprehensive list of public keys, authentication methods, and service entries.

<a id="rationale"></a>
## Rationale

ShareLedger uses built-in account abstraction for on-chain interactions, ensuring that `info.sender` is the verified transaction sender, whether the account is a smart contract or a key pair.

However, because all transactions require funding, there is an increasing demand for "meta-transactions" signed by an external party rather than the transaction originator. This facilitates third-party gas funding or "receiver-pays" models without altering the core architecture. Since these transactions require a key-pair signature, they cannot inherently represent smart-contract-based accounts.

CW1056 provides a mechanism for both smart contracts and regular key pairs to delegate signing authority to external keys for specific purposes. This enables identities to be represented seamlessly both on-chain and off-chain (e.g., in payment channels) through temporary or permanent delegates.
