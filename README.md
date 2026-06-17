# QSB DID - `did:qsb` Method Specification

## Status

This document describes the current `did:qsb` method behavior implemented by
the DID pallet in `Quantum-Blockchains/qsb-node` on `main` as inspected at
commit `2bdb50360fc21ed4ece841b122f7d9368bd5b15c`.

The runtime stores canonical DID state on-chain as `DidDetails`. DID Resolution
output is derived from that state by resolver logic.

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and
"MAY" in this document are to be interpreted as described in RFC 2119 and
RFC 8174.

## DID Syntax

Canonical DID syntax:

```text
did:qsb:<did-id>
```

Where `did-id` is Base58 encoding of exactly 32 bytes.

Runtime and resolver interfaces that accept DID identifiers MUST require full
`did:qsb:<did-id>` syntax. Inputs that do not start with `did:qsb:` are invalid.

## DID Identifier Derivation

The DID identifier is derived from the decoded raw public key of the creation
key:

```text
material = "QSB_DID" || genesis_hash_32_bytes || raw_public_key
did_id_bytes = blake2_256(material)
did-id = base58(did_id_bytes)
did = "did:qsb:" || did-id
```

Notes:

- `blake2_256` is Substrate `sp_io::hashing::blake2_256`.
- `genesis_hash_32_bytes` is `frame_system::Pallet::<T>::block_hash(0)` and
  provides chain separation.
- `create_did` derives the DID from the raw ML-DSA-44 public key decoded from
  the submitted Multikey.

## Key Material

The runtime supports two key material input formats for non-creation key
management:

```rust
enum KeyMaterialInput {
    Multikey(Vec<u8>),
    Jwk(Vec<u8>),
}
```

Normalized on-chain key material is stored as:

```rust
enum DidKeyMaterial {
    Multikey {
        multicodec: u64,
        public_key: Vec<u8>,
    },
    Jwk {
        public_key_jwk: Vec<u8>,
    },
}
```

### Multikey

Runtime validates Multikey input by requiring:

- multibase prefix `u`,
- Base64URL without padding decodability,
- leading unsigned varint multicodec,
- non-empty raw key bytes after the multicodec prefix.

Policy:

- `create_did` MUST accept only Multikey ML-DSA-44 (`0x1210`).
- Rotation of the `#update` key MUST accept only Multikey ML-DSA-44
  (`0x1210`).
- `add_key` and `rotate_key` validate Multikey syntax and enforce raw key
  lengths for known codecs.
- Unknown Multikey codecs are allowed by `add_key` and non-`#update`
  `rotate_key` if the Multikey syntax is valid.

Known codecs with runtime length checks:

- `0x1210` ML-DSA-44: 1312 bytes
- `0x1211` ML-DSA-65: 1952 bytes
- `0x1212` ML-DSA-87: 2592 bytes
- `0x00ED` Ed25519 public key: 32 bytes
- `0x1200` secp256k1 public key: 33 or 65 bytes
- `0x1201` P-256 public key: 33 or 65 bytes

### JWK

JWK key material is stored as an opaque byte sequence:

- it MUST be non-empty,
- it MUST NOT exceed runtime `MaxJwkLength`,
- it is not parsed or semantically validated as JSON by the runtime,
- it MUST NOT be assigned `CapabilityInvocation`,
- it MUST NOT be used as the `#update` key.

In DID Document output, JWK key material maps to `JsonWebKey2020` with
`publicKeyJwk`.

## On-Chain DID State

Per DID, runtime stores:

```rust
struct DidDetails {
    version: u64,
    deactivated: bool,
    keys: Vec<DidKey>,
    services: Vec<ServiceEndpoint>,
    metadata: Vec<MetadataEntry>,
    next_key_index: u32,
}
```

Each DID key is:

```rust
struct DidKey {
    key_id: Vec<u8>,
    key_material: DidKeyMaterial,
    roles: Vec<KeyRole>,
    controller: Option<Vec<u8>>,
    revoked: bool,
}
```

`key_id` MUST be a full DID URL, for example:

```text
did:qsb:<did-id>#update
did:qsb:<did-id>#key-2
```

`controller`, if present, MUST be a strict DID URI:

```text
did:qsb:<did-id>
```

It MUST NOT contain a path, query, or fragment.

## Key Roles

Supported key roles:

- `Authentication`
- `AssertionMethod`
- `KeyAgreement`
- `CapabilityInvocation`
- `CapabilityDelegation`

Only the active `#update` key authorizes DID state mutations.

## Authorization Model

DID mutation authorization is bound to this fixed update key id:

```text
did:qsb:<did-id>#update
```

Rules:

- All mutation-call DID signatures MUST verify against the active, non-revoked
  `#update` key.
- The `#update` key MUST have role `CapabilityInvocation`.
- The `#update` key MUST be Multikey ML-DSA-44.
- Additional keys MAY carry `Authentication`, `AssertionMethod`,
  `KeyAgreement`, `CapabilityInvocation`, or `CapabilityDelegation`, subject to
  key material restrictions.
- Additional keys do not authorize state mutation unless their key id is the
  fixed `#update` key.
- `revoke_key` MUST reject revocation of `#update`.
- `rotate_key` for `#update` MUST preserve `#update` as the new key id.
- `rotate_key` for `#update` MUST enforce Multikey ML-DSA-44 (`0x1210`) for the
  new key material.
- `rotate_key` for `#update` MUST enforce `CapabilityInvocation` in the new
  roles.
- `update_roles` for `#update` MUST preserve `CapabilityInvocation`.
- Runtime signature verification MUST reject invalid `#update` state, including
  missing key, revoked key, missing `CapabilityInvocation`, non-Multikey key
  material, or invalid ML-DSA-44 public key material.

## DID Creation

Wallet flow:

1. Generate an ML-DSA-44 keypair.
2. Build a Multikey value for the public key.
3. Build payload:

   ```text
   "QSB_DID_CREATE" || SCALE(multikey_bytes)
   ```

4. Sign the payload with the ML-DSA-44 private key.
5. Submit:

   ```text
   create_did(multikey_bytes, did_signature)
   ```

Runtime flow:

1. Validate Multikey syntax.
2. Enforce codec `0x1210` (ML-DSA-44).
3. Decode the raw public key.
4. Verify the DID signature with the decoded raw public key.
5. Derive DID from raw key plus genesis hash.
6. Store initial key as `did:qsb:<did-id>#update` with role
   `CapabilityInvocation`.
7. Initialize `version = 0`, `deactivated = false`, empty `services`, empty
   `metadata`, and `next_key_index = 2`.

## Extrinsics

All DID extrinsics require a signed Substrate origin and a DID-level signature
where specified.

### `create_did`

```text
create_did(public_key, did_signature)
```

Parameters:

- `public_key: Vec<u8>` - Multikey ML-DSA-44
- `did_signature: Vec<u8>`

Payload:

```text
"QSB_DID_CREATE" || SCALE(public_key)
```

Effects:

- creates a DID record,
- stores the initial `#update` key,
- emits `DidCreated { did }`.

### `add_key`

```text
add_key(did_id, key_id_suffix, key_material, roles, controller, did_signature)
```

Parameters:

- `did_id: Vec<u8>` - full DID string
- `key_id_suffix: Option<Vec<u8>>`
- `key_material: KeyMaterialInput`
- `roles: Vec<KeyRole>`
- `controller: Option<Vec<u8>>`
- `did_signature: Vec<u8>`

Payload:

```text
"QSB_DID_ADD_KEY"
  || SCALE(did_id)
  || SCALE(key_id_suffix)
  || SCALE(key_material)
  || SCALE(roles)
  || SCALE(controller)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `key_material` MUST be unique after normalization.
- Multikey input is normalized to `DidKeyMaterial::Multikey`.
- JWK input is normalized to `DidKeyMaterial::Jwk`.
- JWK input MUST NOT be assigned `CapabilityInvocation`.
- `controller`, if present, MUST be a strict DID URI.
- `key_id_suffix` MAY be provided as `key-1` or `#key-1`.
- Full DID values in `key_id_suffix` MUST be rejected.
- If `key_id_suffix` is absent, runtime auto-generates `#key-N` using
  `next_key_index`.

Effects:

- appends a new active `DidKey`,
- increments `version`,
- emits `KeyAdded { did, key_material }`.

### `revoke_key`

```text
revoke_key(did_id, key_id, did_signature)
```

Payload:

```text
"QSB_DID_REVOKE_KEY" || SCALE(did_id) || SCALE(key_id)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `key_id` MUST identify an existing non-revoked key.
- `key_id` MUST NOT be the fixed `#update` key.

Effects:

- marks the key as revoked,
- increments `version`,
- emits `KeyRevoked { did, key_material }`.

### `deactivate_did`

```text
deactivate_did(did_id, did_signature)
```

Payload:

```text
"QSB_DID_DEACTIVATE" || SCALE(did_id)
```

Rules:

- DID MUST exist.
- DID MUST NOT already be deactivated.

Effects:

- sets `deactivated = true`,
- increments `version`,
- emits `DidDeactivated { did }`.

Deactivation is irreversible.

### `add_service`

```text
add_service(did_id, service, did_signature)
```

`ServiceEndpoint`:

```rust
struct ServiceEndpoint {
    id: Vec<u8>,
    service_type: Vec<u8>,
    endpoint: Vec<u8>, // serialized as serviceEndpoint
}
```

Payload:

```text
"QSB_DID_ADD_SERVICE" || SCALE(did_id) || SCALE(service)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `service.id` MUST be unique for the DID.
- `service.id` MUST be either a valid fragment (`#...`) or an absolute URI.
- `service.endpoint` MUST be an absolute URI.

Effects:

- appends the service,
- increments `version`,
- emits `ServiceAdded { did, service_id }`.

### `remove_service`

```text
remove_service(did_id, service_id, did_signature)
```

Payload:

```text
"QSB_DID_REMOVE_SERVICE" || SCALE(did_id) || SCALE(service_id)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `service_id` MUST identify an existing service.

Effects:

- removes the service,
- increments `version`,
- emits `ServiceRemoved { did, service_id }`.

### `set_metadata`

```text
set_metadata(did_id, entry, did_signature)
```

`MetadataEntry`:

```rust
struct MetadataEntry {
    key: Vec<u8>,
    value: Vec<u8>,
}
```

Payload:

```text
"QSB_DID_SET_METADATA" || SCALE(did_id) || SCALE(entry)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.

Effects:

- updates `entry.key` if it already exists, otherwise inserts a new metadata
  entry,
- increments `version`,
- emits `MetadataSet { did, key }`.

### `remove_metadata`

```text
remove_metadata(did_id, key, did_signature)
```

Payload:

```text
"QSB_DID_REMOVE_METADATA" || SCALE(did_id) || SCALE(key)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `key` MUST identify an existing metadata entry.

Effects:

- removes the metadata entry,
- increments `version`,
- emits `MetadataRemoved { did, key }`.

### `rotate_key`

```text
rotate_key(
  did_id,
  old_key_id,
  new_key_material,
  new_key_id_suffix,
  new_controller,
  roles,
  did_signature
)
```

Parameters:

- `did_id: Vec<u8>` - full DID string
- `old_key_id: Vec<u8>` - full DID URL of the key being rotated
- `new_key_material: KeyMaterialInput`
- `new_key_id_suffix: Option<Vec<u8>>`
- `new_controller: Option<Vec<u8>>`
- `roles: Vec<KeyRole>`
- `did_signature: Vec<u8>`

Payload:

```text
"QSB_DID_ROTATE_KEY"
  || SCALE(did_id)
  || SCALE(old_key_id)
  || SCALE(new_key_material)
  || SCALE(new_key_id_suffix)
  || SCALE(new_controller)
  || SCALE(roles)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `old_key_id` MUST identify an existing non-revoked key.
- `new_key_material` MUST be unique after normalization.
- If `old_key_id` is `#update`, the new key id MUST remain `#update`.
- If `old_key_id` is `#update`, `new_key_material` MUST be Multikey ML-DSA-44.
- If `old_key_id` is `#update`, `roles` MUST include `CapabilityInvocation`.
- For non-`#update` rotation, key id suffix rules are the same as `add_key`.
- `new_controller`, if present, MUST be a strict DID URI.
- JWK input MUST NOT be assigned `CapabilityInvocation`.

Effects:

- marks the old key as revoked,
- appends a new active key,
- increments `version`,
- emits `KeyRotated { did, old_key_material, new_key_material }`.

### `update_roles`

```text
update_roles(did_id, key_id, roles, did_signature)
```

Payload:

```text
"QSB_DID_UPDATE_ROLES" || SCALE(did_id) || SCALE(key_id) || SCALE(roles)
```

Rules:

- DID MUST exist and MUST NOT be deactivated.
- `key_id` MUST identify an existing non-revoked key.
- If `key_id` is `#update`, `roles` MUST include `CapabilityInvocation`.
- JWK keys MUST NOT be assigned `CapabilityInvocation`.

Effects:

- replaces the key role set,
- increments `version`,
- emits `RolesUpdated { did, key_material }`.

## Key ID Suffix Rules

For `add_key` and non-`#update` `rotate_key`:

- `key_id_suffix` MAY be omitted.
- If omitted, runtime generates `#key-N` using `next_key_index`.
- If provided without `#`, runtime prefixes it with `#`.
- The normalized suffix MUST start with `#` and contain at least one character
  after `#`.
- The normalized suffix MUST NOT contain a second `#`.
- Full DID values such as `did:qsb:...#key-1` MUST be rejected.
- The resulting full key id MUST be unique within the DID record.

## DID Resolution Output

Resolver logic returns a DID Resolution result:

- `didDocument`
- `didDocumentMetadata`
- `didResolutionMetadata`

Successful resolution:

- `didResolutionMetadata.contentType = "application/did+ld+json"`
- `didResolutionMetadata.error = null`
- `didDocument` is populated
- `didDocumentMetadata.deactivated` mirrors on-chain state
- `didDocumentMetadata.versionId` mirrors on-chain `version`

Resolution errors:

- malformed DID input:
  - `didResolutionMetadata.error = "invalidDid"`
  - `didDocument = null`
- syntactically valid but absent DID:
  - `didResolutionMetadata.error = "notFound"`
  - `didDocument = null`

### DID Document Mapping

The DID Document contains:

- `@context`
- `id`
- `verificationMethod`
- `authentication`
- `assertionMethod`
- `keyAgreement`
- `capabilityInvocation`
- `capabilityDelegation`
- `service`

Revoked keys MUST be excluded from active DID Document verification
relationships.

Verification method mapping:

- Multikey material maps to:
  - `type = "Multikey"`
  - `publicKeyMultibase`
- JWK material maps to:
  - `type = "JsonWebKey2020"`
  - `publicKeyJwk`

`verificationMethod[].id` MUST be a DID URL. `verificationMethod[].controller`
MUST be either the stored strict DID controller or the DID itself.

## Runtime API and RPC

Runtime API:

```text
did_by_string(did: Vec<u8>) -> Option<DidDetails>
```

Node RPC:

```text
did_getByString
```

The RPC returns raw on-chain `DidDetails`. DID Resolution output is derived by
resolver logic, not by the node RPC.

## QSB Network Connectivity

Known QSB RPC endpoints:

| Network | RPC WebSocket |
| --- | --- |
| testnet | `wss://qsb.qbck.io:9945` |

Resolvers and wallets targeting QSB testnet SHOULD connect to the testnet RPC
WebSocket endpoint above and use node RPC `did_getByString` for raw DID state
lookup.

## Reference Resolver

Reference resolver (`qsb-did-resolver`) provides:

- HTTP endpoint: `GET /1.0/identifiers/{did}`
- DID syntax validation for `did:qsb:...`
- mapping from `did_getByString` raw state to DID Resolution response

Runtime and node expose raw DID state. Canonical DID Resolution output is the
resolver's responsibility.

## Security and Privacy

- On-chain DID state is public and permanent.
- Private keys MUST remain off-chain.
- Service endpoints and metadata may leak sensitive data.
- JWK bytes are stored opaquely and are not validated as JSON by runtime.
- Compromised update keys SHOULD be rotated immediately.
- Loss of the active `#update` private key can make DID mutation impossible.
