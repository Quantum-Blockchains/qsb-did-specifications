# QSB DID — Specification of the `did:qsb` Method

## Description

The `did:qsb` method is a native, on-chain Decentralized Identifier (DID) method operating on the QSB Blockchain, built on the Substrate framework. It provides post‑quantum resistance by using the ML‑DSA‑44 algorithm (NIST FIPS 204) as its primary cryptographic mechanism. The entire DID state is stored directly on-chain and is deterministically resolved from chain state. For background, see the Quantum Blockchain whitepaper: https://www.quantumblockchains.io/introducing-quantum-secured-blockchain-a-comprehensive-whitepaper/

## DID Syntax

```
did:qsb:<did-id>
```

The identifier does not contain explicit network information. Network separation is ensured by the derivation of `did-id`, which incorporates the genesis value of the given network.

## DID Identifier Derivation

Each DID is deterministically derived from an ML‑DSA‑44 public key.

Algorithm:

```
material = "QSB_DID" || genesis || pk
did_id_bytes = blake2_256(material)
did-id = base58(did_id_bytes)
did = "did:qsb:" || did-id
```

Notes:

- ML‑DSA‑44 keys are generated on the user side.
- The private key never leaves the user’s device.
- Only public keys are stored on-chain.
- The on-chain address (account) is used solely to pay transaction fees and does not control the identity.

## On-Chain DID State

For each DID, the following data are stored:

- version counter
- deactivation flag
- ML‑DSA‑44 keys with assigned roles
- service endpoints (optional)
- metadata

## Key Roles

Supported key roles:

- `Authentication` — authentication of the DID controller.
- `AssertionMethod` — signing of statements and attestations.
- `KeyAgreement` — key agreement and encryption.
- `CapabilityInvocation` — performing operations on behalf of the DID.
- `CapabilityDelegation` — delegating authority to other keys or entities.

## DID Creation

Wallet side:

- Generate an ML‑DSA‑44 key pair.
- Build the payload: `"QSB_DID_CREATE" || pk`
- Sign the payload with the ML‑DSA‑44 private key.
- Submit the transaction `create_did(pk, did_signature)`, signed by the fee‑paying account.

Runtime side:

- Verify the DID signature.
- Compute the `did-id`.
- Persist the DID state in chain storage.
- Emit the event `DidCreated(did)`.

## On-Chain Functions

- `create_did(public_key, did_signature)` — registers a DID with an ML‑DSA‑44 public key.
- `add_key(did_id, public_key, roles, did_signature)` — adds a new public key (rotation / additional key) and assigns roles.
- `revoke_key(did_id, public_key, did_signature)` — marks the specified key as revoked.
- `deactivate_did(did_id, did_signature)` — permanently deactivates the DID (no further changes are allowed after deactivation).
- `add_service(did_id, service, did_signature)` — adds a service endpoint to the DID.
  - `service.id` — service identifier.
  - `service.service_type` — service type.
  - `service.endpoint` — endpoint address.
- `remove_service(did_id, service_id, did_signature)` — removes a service endpoint from the DID.
- `set_metadata(did_id, entry, did_signature)` — sets or updates a metadata entry.
  - `entry.key` — metadata key.
  - `entry.value` — metadata value.
- `remove_metadata(did_id, key, did_signature)` — removes a metadata entry.
- `rotate_key(did_id, old_public_key, new_public_key, roles, did_signature)` — rotates a key (revokes the old key and adds the new key with roles).
- `update_roles(did_id, public_key, roles, did_signature)` — updates roles assigned to a key.

## Deactivation and Revocation Policy

- DID deactivation is irreversible.
- After deactivation, no further updates to the DID state are permitted.
- Revoked keys remain in the on-chain state as `revoked`.

## Privacy & Security Considerations

- All on-chain data are public, permanent, and readable by anyone.
- Only public keys are stored on-chain; private keys must be generated, stored, and protected by the user.
- Service endpoints and metadata may reveal sensitive information and should be used with care.
- Key compromise requires immediate revocation or rotation to reduce risk.
- Revoked keys remain visible in state history and should be treated as invalid by resolvers.

## Retrieving Data for a DID Document

Data required to build a DID Document can be retrieved in two ways:

1. Directly from storage:
Read the entry from the `DidRecords` map using the `did-id` key (32 bytes). First decode `did:qsb:<did-id>` to a 32‑byte identifier, then fetch `DidDetails` from state storage.

2. Via Runtime API:
Call the runtime method that accepts the DID as a byte string (`did:qsb:<did-id>`) and returns `DidDetails`. This allows a resolver to retrieve data without direct storage access.

## DID Document Example

```
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "did:qsb:<did-id>",
  "controller": [
    "did:qsb:<did-id>"
  ],
  "authentication": [
    "did:qsb:<did-id>#key-1"
  ],
  "verificationMethod": [
    {
      "id": "did:qsb:<did-id>#key-1",
      "type": "<type>",
      "controller": "did:qsb:<did-id>",
      "publicKeyHex": "<pk>"
    }
  ],
  "service": [
    {
      "id": "did:qsb:<did-id>#service-1",
      "type": "<service-type>",
      "serviceEndpoint": "<service-endpoint>"
    }
  ],
  "version": "1.0",
  "deactivated": false
}
```
