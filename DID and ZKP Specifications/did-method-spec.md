---
title: SHR DID Method Specification
post_type: page
menu_order: 1
post_status: publish
post_excerpt: The formal specification for the did:shr decentralized identifier method.
taxonomy:
    category:
        - identity
---

# SHR DID Method Specification

## Table of Contents

1. [Preface](#preface)
2. [Abstract](#abstract)
   - [Identifier Controller](#identifier-controller)
   - [Target System](#target-system)
   - [Advantages](#advantages)
3. [Technical Specifications](#technical-specifications)
   - [JSON-LD Context Definition](#json-ld-context-definition)
   - [DID Method Name](#did-method-name)
   - [Method Specific Identifier](#method-specific-identifier)
   - [Relationship to VCT](#relationship-to-vct)
4. [CRUD Operation Definitions](#crud-operation-definitions)
   - [Create (Register)](#create-register)
   - [Read (Resolve)](#read-resolve)
     - [Controller Address](#controller-address)
     - [Enumerating Contract Events to build the DID Document](#enumerating-contract-events-to-build-the-did-document)
     - [Controller changes (`change_did_owner`)](#controller-changes-change_did_owner)
     - [Delegate Keys (`change_did_delegate`)](#delegate-keys-change_did_delegate)
     - [Non-Shareledger Attributes (`DIDAttributeChanged`)](#non-shareledger-attributes-didattributechanged)
     - [Public Keys](#public-keys)
     - [Service Endpoints](#service-endpoints)
     - [id properties of entries](#id-properties-of-entries)
   - [Update](#update)
   - [Delete (Revoke)](#delete-revoke)
5. [Metadata](#metadata)
   - [DID Document Metadata](#did-document-metadata)
   - [DID Resolution Metadata](#did-resolution-metadata)
6. [Resolving DID URIs with Query Parameters](#resolving-did-uris-with-query-parameters)
   - [versionId query string parameter](#versionid-query-string-parameter)
   - [Security considerations of DID versioning](#security-considerations-of-did-versioning)
   - [initial-state query string parameter](#initial-state-query-string-parameter)
7. [Reference Implementations](#reference-implementations)
8. [References](#references)

<a id="preface"></a>
## Preface

The **SHR DID method specification** adheres to the requirements outlined in the W3C DID specification, which is currently published by the W3C Credentials Community Group. It is largely derived from the **ETHR DID method specification**, with specific adjustments made to accommodate the unique requirements of the ShareRing ecosystem. This document outlines the technical standards for decentralized identifiers on ShareLedger.

<a id="abstract"></a>
## Abstract

<a id="identifier-controller"></a>
### Identifier Controller

By default, each identifier is controlled by itself—or more accurately, by its corresponding private key. This means that a single public key controls the identifier at any given time. However, the controller has the flexibility to replace themselves with any other public key. This capability extends to contracts, enabling more advanced control models such as multi-signature governance.

<a id="target-system"></a>
### Target System

The target system for this method is the **ShareLedger network**, specifically where the **CW1056** smart contract registry is deployed. This registry serves as the on-chain anchor for all identity operations.

<a id="advantages"></a>
### Advantages

This method offers several key benefits designed to optimize efficiency and privacy:

-   **Zero-Cost Creation:** No transaction fees are required to create an identifier.
    
-   **Privacy:** Identifier creation is performed locally and remains private until an on-chain interaction occurs.
    
-   **Account Abstraction:** It fully leverages ShareLedger's built-in account abstraction capabilities.
    
-   **Flexible Control:** It supports multi-sig or proxy wallets acting as account controllers.
    
-   **Standardized Keys:** It natively supports **secp256k1** public keys as identifiers.
    
-   **Data Decoupling:** It decouples claims data and ShareLedger interactions from the underlying identifier, enhancing modularity.
    
-   **Gas Flexibility:** It allows third-party funding services to pay gas fees via meta-transactions ("receiver pays").
    
-   **Interoperability:** It is compatible with any CosmWasm-compliant blockchain.
    
-   **Versioning:** It supports native, verifiable versioning of DID documents.

<a id="technical-specifications"></a>
## Technical Specifications

<a id="json-ld-context-definition"></a>
### JSON-LD Context Definition

Because this DID method supports both `publicKeyHex` and `publicKeyBase64` encodings for verification methods, it requires a valid JSON-LD context for those entries. To ensure proper JSON-LD processing, the `@context` used when constructing DID documents for `did:shr` should be defined as follows:

```JSON
"@context": [
  "https://www.w3.org/ns/did/v1",
  "https://w3id.org/security/suites/secp256k1recovery-2020/v2"
]
```

You must include this `@context` if you intend to use `EcdsaSecp256k1RecoveryMethod2020` in your applications.

<a id="did-method-name"></a>
### DID Method Name

The method name is `shr`.

<a id="method-specific-identifier"></a>
### Method Specific Identifier

The method-specific identifier is represented as the **HEX-encoded secp256k1 public key** (in compressed form), or the corresponding **HEX-encoded ShareLedger address** on the target network, prefixed with `shareledger`.

**Syntax:**

Plaintext

```
shr-did = "did:shr:" shr-specific-identifier
shr-specific-identifier = [ shr-token-type ":" ] public-key-hex
shr-token-type = "shr" /"dvct" / "sbt" 
public-key-hex = "0x" 66*HEXDIG
```

**Notes:**

-   `“shr”` refers to the VCT token type.
    
-   The public key must strictly follow these rules:
    
    1.  It **MUST** be represented in compressed form (refer to the [Secp256k1 Wiki](https://en.bitcoin.it/wiki/Secp256k1)).
        
    2.  The corresponding `blockchainAccountId` entry is automatically added to the default DID document, unless the owner property has been changed to a different address.
        
    3.  All **Read**, **Update**, and **Delete** operations **MUST** be made using the corresponding `blockchainAccountId` and **MUST** originate from the correct controller account (the CW1056 owner).
        
    4.  In the case of an issuer, the issuer will be the owner of the DID.

<a id="crud-operation-definitions"></a>
## Relationship to VCT - CRUD Operation Definitions

<a id="create-register"></a>
### Create (Register)

To create a `shr` DID, a public key (i.e., a key pair) needs to be generated locally. At this stage, no interaction with the target ShareLedger blockchain is required. Registration is **implicit**: it is computationally impossible to brute-force a public key (i.e., guess the private key for a given public key on the secp256k1 curve). The holder of the private key is inherently the entity identified by the DID.

The default DID document for a `did:shr:<public key>`—for example, `did:shr:02d04758db085b20793240eddc393c8850ec2587cb57d8f9baf1acbb6f34cbabc7`—with no prior transactions to the CW1056 registry looks like this:

```JSON
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/secp256k1recovery-2020/v2"
  ],
  "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
  "verificationMethod": [
    {
      "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#controller",
      "type": "EcdsaSecp256k1RecoveryMethod2020",
      "controller": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
      "blockchainAccountId": "cosmos:shareledger:1zq5e4w2w552t4slmclezsuxu5jupmpu52ut6vd"
    },
     {
      "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#controllerKey",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "controller": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
      "publicKeyHex": "02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9"
    }
    
  ],
  "authentication": [
    "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#controller"
  ],
  "assertionMethod": [
    "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#controller"
  ]
}
```

<a id="read-resolve"></a>
### Read (Resolve)

The DID document is dynamically built by querying read-only functions and interpreting contract events from the CW1056 registry. Additional verification relationships and service entries are added or removed by enumerating these contract events.

<a id="controller-address"></a>
#### Controller Address

Each identifier always has a controller address. The resolver **MUST** first check the read-only contract function `identityOwner(address identity)` on the deployed CW1056 contract to determine the current owner.

This controller address **MUST** be represented in the DID document as a `verificationMethod` entry. The `id` is set as the DID being resolved with the fragment `#controller` appended. A reference to this entry **MUST** also be added to both the `authentication` and `assertionMethod` arrays of the DID document.

<a id="enumerating-contract-events-to-build-the-did-document"></a>
#### Enumerating Contract Events to build the DID Document

The CW1056 contract publishes three specific types of events for each identifier:

1.  `change_did_owner` action (indicating a change of controller)
    
2.  `change_did_delegate` action
    
3.  `change_did_attribute` action
    

If a change has ever been made for the ShareLedger address of an identifier, the block number of the most recent change is stored in the contract's `changed` mapping. The latest event can be efficiently located by checking for one of the three event types at that exact block height. Each CW1056 event also contains a `previous_change` value, which points to the block number of the preceding change (if any), creating a linked list of history.

To retrieve the full history of changes for an address, use the following logic:

1.  Query `changed(String identity)` on the CW1056 contract to get the latest block where a change occurred.
    
2.  If the result is null, return (no changes).
    
3.  Filter for the above event types emitted by the contract address at that specified block.
    
4.  If the event has a `previous_change` value, repeat step 3 for that block number.
    

After reconstructing the history of events for an address, interpret each event to build the DID document as follows:

<a id="controller-changes-change_did_owner"></a>
#### Controller changes (`change_did_owner`)

When the controller address of a `did:shr` is updated, a `change_did_owner` action event is emitted. The data from this event **MUST** be used to update the `#controller` entry in the `verificationMethod` array. When resolving DIDs with `publicKey` identifiers, if the controller (owner) address differs from the address corresponding to the `publicKey`, then the `#controllerKey` entry in the `verificationMethod` array **MUST** be omitted.

```JSON
[
    {
        "events": [
            {
                "type": "wasm",
                "attributes": [
                    {
                        "key": "_contract_address",
                        "value": "shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm"
                    },
                    {
                        "key": "action",
                        "value": "change_did_owner"
                    },
                    {
                        "key": "owner",
                        "value": "shareledger10kwzf5eq0fz83kxtvw2sqed3qktz2834t4kyxezh2vgfsj2cnkqqzkaxa3"
                    },
                    {
                        "key": "previous_change",
                        "value": "1000"
                    }
                ]
            }
        ],
        "msg_index": 0
    }
]
```

<a id="delegate-keys-change_did_delegate"></a>
#### Delegate Keys (`change_did_delegate`)

Delegate keys are ShareLedger or ICA addresses that can serve as general signing keys or optionally perform authentication. These delegates are verifiable directly from the deployed CW1056 smart contract.

When a delegate is added or revoked, a `change_did_delegate` event is published. This event **MUST** be used to update the DID document.

-   **identity:** The ShareLedger address that is being delegated to.
    
-   **delegate_types:** There are two types currently published in the DID document:
    
    -   **veriKey:** This adds a `EcdsaSecp256k1RecoveryMethod2020` to the `verificationMethod` section of the DID document with the `blockchainAccountId(shareledgerAddress)` of the delegate. It also adds a reference to it in the `assertionMethod` section.
        
    -   **sigAuth:** This adds a `EcdsaSecp256k1RecoveryMethod2020` to the `verificationMethod` section of the document and a reference to it in the `authentication` section.
        

Only events with a `validTo` timestamp (measured in seconds) greater than or equal to the current time should be included in the DID document. When resolving an older version (using `versionId` in the `didURL` query string), the `validTo` entry **MUST** be compared against the timestamp of the block at `versionId` height.

Such valid delegates **MUST** be added to the `verificationMethod` array as `EcdsaSecp256k1RecoveryMethod2020` entries, with the delegate address listed in the `blockchainAccountId` property and prefixed with `cosmos:shareledger:`, in accordance with CAIP10.

**Example:**

```JSON
[
    {
        "events": [
            {
                "type": "wasm",
                "attributes": [
                    {
                        "key": "_contract_address",
                        "value": "shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm"
                    },
                    {
                        "key": "action",
                        "value": "change_did_delegate"
                    },
                    {
                        "key": "identity",
                        "value": "shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm"
                    },
                    {
                        "key": "delegate_type",
                        "value": "veriKey"
                    },
                    {
                        "key": "valid_to",
                        "value": "2000"
                    },
                    {
                        "key": "previous_change",
                        "value": "1000"
                    }
                ]
            }
        ],
        "msg_index": 0
    }
]
```

<a id="non-shareledger-attributes-didattributechanged"></a>
#### Non-Shareledger Attributes (`DIDAttributeChanged`)

Attributes allow for the addition of non-ShareLedger keys, service endpoints, and other metadata. Attributes exist solely on the blockchain as contract events and cannot be queried directly from within CosmWasm code.

**name:**

```JSON
[
    {
        "events": [
            {
                "type": "wasm",
                "attributes": [
                    {
                        "key": "_contract_address",
                        "value": "shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm"
                    },
                    {
                        "key": "action",
                        "value": "change_did_delegate"
                    },
                    {
                        "key": "identity",
                        "value": "shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm"
                    },
                    {
                        "key": "delegate_type",
                        "value": "veriKey"
                    },
                    {
                        "key": "previous_change",
                        "value": "1000"
                    }u
                ]
            }
        ],
        "msg_index": 0
    }
]
```

While any attribute can be stored, for the purpose of the DID document, we strictly support adding to the following sections:

-   Public Keys (Verification Methods)
    
-   Service Endpoints
    

This design decision is intentional, aiming to discourage the use of custom attributes in DID documents to prevent the misuse of the chain for storing personal user information.

<a id="public-keys"></a>
#### Public Keys

The name of the attribute added to CW1056 should follow this format: `did/pub/(Secp256k1|RSA|Ed25519|X25519)/(veriKey|sigAuth|enc)/(hex|base64|base58)`

(Essentially: `did/pub/<key algorithm>/<key purpose>/<encoding>`)

Please opt for the `base58` encoding, as other encodings are not spec-compliant and will be removed in future versions of the spec and reference resolver.

**Key purposes**

-   **veriKey:** Adds a verification key to the `verificationMethod` section of the document and adds a reference to it in the `assertionMethod` section.
    
-   **sigAuth:** Adds a verification key to the `verificationMethod` section of the document and adds a reference to it in the `authentication` section.
    
-   **enc:** Adds a key agreement key to the `verificationMethod` section and a corresponding entry to the `keyAgreement` section. This key is used to perform a Diffie-Hellman key exchange and derive a secret key for encrypting messages to the DID that lists such a key.
    

> **Note:** The `<encoding>` refers only to the key encoding in the resolved DID document. Attribute values sent to the CW1056 registry should always be hex encodings of the raw public key data.

**Example: Hex encoded Secp256k1 Verification Key**

A `DIDAttributeChanged` event for the account `shareledger1zq5e4w2w552t4slmclezsuxu5jupmpu52ut6vd` with the name `did/pub/Secp256k1/veriKey/hex` and the value `02b97c30de767f084ce3080168ee293053ba33b235d7116a3263d29f1450936b71` generates a verification method entry like the following:

```JSON
{
  "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#delegate-1",
  "type": "EcdsaSecp256k1VerificationKey2019",
  "controller": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
  "publicKeyHex": "02b97c30de767f084ce3080168ee293053ba33b235d7116a3263d29f1450936b71"
}
```

**Example: Base58 encoded Ed25519 Verification Key**

A `DIDAttributeChanged` event for the account `shareledger1zq5e4w2w552t4slmclezsuxu5jupmpu52ut6vd` with the name `did/pub/Ed25519/veriKey/base58` and the value `b97c30de767f084ce3080168ee293053ba33b235d7116a3263d29f1450936b71` generates a verification method entry like this:

```JSON
{
  "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#delegate-1",
  "type": "Ed25519VerificationKey2018",
  "controller": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
  "publicKeyBase58": "DV4G2kpBKjE6zxKor7Cj21iL9x9qyXb6emqjszBXcuhz"
}
```

**Example: Base64 encoded X25519 Encryption Key**

A `DIDAttributeChanged` event for the account `shareledger1zq5e4w2w552t4slmclezsuxu5jupmpu52ut6vd` with the name `did/pub/X25519/enc/base64` and the value `302a300506032b656e032100118557777ffb078774371a52b00fed75561dcf975e61c47553e664a617661052` generates a verification method entry like this:

```JSON
{
  "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#delegate-1",
  "type": "X25519KeyAgreementKey2019",
  "controller": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9",
  "publicKeyBase64": "MCowBQYDK2VuAyEAEYVXd3/7B4d0NxpSsA/tdVYdz5deYcR1U+ZkphdmEFI="
}
```

<a id="service-endpoints"></a>
#### Service Endpoints

The name of the attribute should follow this format: `did/svc/[ServiceName]`

**Example:**

A `DIDAttributeChanged` event for the account `shareledger1lfhglvsc425mhdlzv582pp5qvaxmgv94v0jgawy9quz2exkvk8qsd4c5lm` with the name `did/svc/HubService` and value of the URL `https://hubs.uport.me` hex encoded as `0x68747470733a2f2f687562732e75706f72742e6d65` generates a service endpoint entry like the following:

```JSON
{
  "id": "did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9#service-1",
  "type": "HubService",
  "serviceEndpoint": "https://auth.sharering.network"
}
```

<a id="id-properties-of-entries"></a>
#### id properties of entries

With the exception of `#controller` and `#controllerKey`, the `id` properties that appear throughout the DID document **MUST** be stable across updates. This ensures that the same key material remains referenced by the same ID even after an update occurs.

-   Attribute or delegate changes that result in `verificationMethod` entries **MUST** set the id to `${did}#delegate-${eventIndex}`.
    
-   Attributes that result in service entries **MUST** set the id to `${did}#service-${eventIndex}`.
    

Here, `eventIndex` is the index of the event that modifies that specific section of the DID document.

**Example**

1.  add key => `#delegate-1` is added
    
2.  add another key => `#delegate-2` is added
    
3.  add delegate => `#delegate-3` is added
    
4.  add service => `#service-1` is added
    
5.  revoke first key => `#delegate-1` gets removed from the DID document; `#delegate-2` and `#delegate-3` remain.
    
6.  add another delegate => `#delegate-5` is added (note: earlier revocation is counted as an event)
    
7.  first delegate expires => `delegate-3` is removed, `#delegate-5` remains intact
    

<a id="update"></a>
### Update

The DID Document may be updated by invoking the relevant smart contract functions as defined by the **CW1056 standard**. These updates can include changes to the account owner, adding delegates, and adding additional attributes. For a comprehensive description, please refer to the detailed **CW1056 documentation**.

Invoking these functions triggers the respective contract events, which are then used to reconstruct the DID Document for a given account as described in the [Enumerating Contract Events to build the DID Document](#enumerating-contract-events-to-build-the-did-document) section.

Additionally, some elements of the DID Document will be revoked automatically when their validity period expires. This applies to both delegates and additional attributes. All attribute and delegate functions trigger the respective events, which are essential for building the DID Document for a given identifier.

<a id="delete-revoke"></a>
### Delete (Revoke)

To revoke a DID, the owner property of the identifier **MUST** be set to `None`. This indicates that the account has no owner—a common approach for invalidation (similar to burning tokens). Once this change is made, it is impossible to make further updates to the DID document; therefore, all pre-existing keys and services **MUST** be considered revoked.

If the intention is to revoke all signatures corresponding to the DID, this option **MUST** be used.

The DID resolution result for a deactivated DID has the following shape:

```JSON
{
  "didDocumentMetadata": {
    "deactivated": true
  },
  "didResolutionMetadata": {
    "contentType": "application/did+ld+json"
  },
  "didDocument": {
    "@context": "https://www.w3.org/ns/did/v1",
    "id": "<the deactivated DID>",
    "verificationMethod": [],
    "assertionMethod": [],
    "authentication": []
  }
}
```

<a id="metadata"></a>
## Metadata

The resolution method returns an object containing the following properties: `didDocument`, `didDocumentMetadata`, and `didResolutionMetadata`.

<a id="did-document-metadata"></a>
### DID Document Metadata

When resolving a DID document that has undergone updates, the details of the latest update **MUST** be listed in the `didDocumentMetadata`.

-   `versionId` **MUST** be the block number of the latest update.
    
-   `updated` **MUST** be the ISO date string of the block time of the latest update (without sub-second resolution).
    

**Example:**

```JSON
{
  "didDocumentMetadata": {
    "versionId": "12090175",
    "updated": "2021-03-22T18:14:29Z"
  }
}
```

<a id="did-resolution-metadata"></a>
### DID Resolution Metadata

```JSON
{
  "didResolutionMetadata": {
    "contentType": "application/did+ld+json"
  }
}
```

<a id="resolving-did-uris-with-query-parameters"></a>
## Resolving DID URIs with Query Parameters

<a id="versionid-query-string-parameter"></a>
### versionId query string parameter

This DID method supports resolving previous versions of the DID document by specifying a `versionId` parameter.

**Example:** `did:shr:02457f5dbfdde10a2b8ad3ed73a6abcdbf4e7c326766e5b9f235b0af0b4846f0d9?versionId=12090175`

The `versionId` corresponds to the block number at which the DID resolution **MUST** be performed. Only CW1056 events prior to or contained within this block number are considered when reconstructing the event history.

If there are any events after that block that mutate the DID, the earliest of them **SHOULD** be used to populate the properties of the `didDocumentMetadata`:

-   `nextVersionId` **MUST** be the block number of the next update to the DID document.
    
-   `nextUpdate` **MUST** be the ISO date string of the block time of the next update (without sub-second resolution).
    

In cases where the DID has had updates prior to or included in the `versionId` block number, the `updated` and `versionId` properties of the `didDocumentMetadata` **MUST** correspond to the latest block prior to the `versionId` query string parameter.

Any timestamp comparisons of `validTo` fields in the event history **MUST** be performed against the `versionId` block timestamp.

**Example:** `?versionId=12101682`

```JSON
{
  "didDocumentMetadata": {
    "versionId": "12090175",
    "updated": "2021-03-22T18:14:29Z",
    "nextVersionId": "12276565",
    "nextUpdate": "2021-04-20T10:48:42Z"
  }
}
```

<a id="security-considerations-of-did-versioning"></a>
### Security considerations of DID versioning

Applications **MUST** take precautions when using versioned DID URIs. Even if a key is compromised and subsequently revoked, it could still be used to issue signatures on behalf of an "older" version of the DID URI. The use of versioned DID URIs is recommended only in limited situations where the timestamp of signatures can be reliably verified, where malicious signatures can be easily revoked, or where applications can afford to check for explicit revocations of keys or signatures. Whenever versioned DIDs are in use, it **SHOULD** be made obvious to users that they are interacting with potentially revoked data.

<a id="initial-state-query-string-parameter"></a>
### initial-state query string parameter

TBD

<a id="reference-implementations"></a>
## Reference Implementations

TBD

<a id="references"></a>
## References

-   [1] W3C DID Core Specification: [https://w3c-ccg.github.io/did-core/](https://w3c-ccg.github.io/did-core/)
    
-   [2] Ethereum EIP-1056: [ethereum/EIPs#1056](https://github.com/ethereum/EIPs/issues/1056)
    
-   [3] ETHR DID Resolver: [https://github.com/decentralized-identity/ethr-did-resolver](https://github.com/decentralized-identity/ethr-did-resolver)
    
-   [4] ETHR DID Registry: [https://github.com/uport-project/ethr-did-registry](https://github.com/uport-project/ethr-did-registry)
    
-   [5] ETHR DID Method Spec: [Reference Documentation](https://github.com/decentralized-identity/ethr-did-resolver/blob/master/doc/did-method-spec.md)
