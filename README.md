# QSB DID — `did:qsb` Method Specification

## Description

`did:qsb` is a native on-chain DID method for the QSB blockchain (Substrate-based).
State is stored in runtime storage and resolved deterministically from chain state.
The method uses ML-DSA-44 as the mandatory DID-creation authentication algorithm.

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" in this
document are to be interpreted as described in RFC 2119 and RFC 8174.

## DID Syntax

`did:qsb:<did-id>`

`did-id` is Base58 of a 32-byte identifier.

Resolver and runtime interfaces that accept DID identifiers MUST require the full
`did:qsb:<did-id>` syntax.

## DID Identifier Derivation

DID is derived from raw ML-DSA-44 public key bytes:

```
material = "QSB_DID" || genesis_hash_32_bytes || raw_mldsa44_public_key
did_id_bytes = blake2_256(material)
did-id = base58(did_id_bytes)
did = "did:qsb:" || did-id
```

Notes:
- `blake2_256` is Substrate `sp_io::hashing::blake2_256`.
- `genesis_hash_32_bytes` is network-specific and provides network separation.
- DID creation MUST derive identifier bytes from the decoded raw ML-DSA-44 public key.

## Key Material Formats

Supported verification method types:
- `Multikey`
- `JsonWebKey2020`

Current runtime behavior:
- `create_did` requires `Multikey` input and enforces `ML-DSA-44` multicodec.
- `add_key` and `rotate_key` accept:
  - `Multikey` (validated and decoded to raw key bytes),
  - `JsonWebKey2020` (size-bounded opaque bytes in runtime).

### Multikey Validation

For `Multikey`, runtime validates:
- multibase prefix `u`,
- Base64URL (no padding) decodability,
- leading unsigned varint multicodec code,
- non-empty decoded key bytes.

Supported multicodecs (current implementation):
- `0x1210` ML-DSA-44 (1312 bytes)
- `0x1220` SLH-DSA-SHA2-128s (32 bytes)
- `0x122c` Falcon-512 (897 bytes)
- `0x122e` SQIsign-1 (65 bytes)

`create_did` MUST reject all Multikey values except `0x1210` (ML-DSA-44).

## On-Chain DID State (`DidDetails`)

Per DID, runtime stores:
- `version: u64`
- `deactivated: bool`
- `keys: Vec<DidKey>`
- `services: Vec<ServiceEndpoint>`
- `metadata: Vec<MetadataEntry>`
- `next_key_index: u32`

`DidKey` fields:
- `key_id`
- `vm_type`
- `public_key` (normalized raw bytes)
- `roles`
- `controller` (optional, strict DID URI; may be self or external DID)
- `revoked`

## Key Roles

- `Authentication`
- `AssertionMethod`
- `KeyAgreement`
- `CapabilityInvocation`
- `CapabilityDelegation`

Runtime invariant:
- DID mutation authorization MUST be verified using the non-revoked key with key id
  `did:qsb:<did-id>#update`.
- Additional keys MAY also have `Authentication` role, but MUST NOT replace the
  `#update` key for mutation authorization.

## DID Creation

Wallet flow:
1. Generate ML-DSA-44 keypair.
2. Build Multikey value for the public key.
3. Build payload: `"QSB_DID_CREATE" || SCALE(multikey_bytes)`.
4. Sign payload with ML-DSA-44 private key.
5. Submit `create_did(multikey_bytes, did_signature)`.

Runtime flow:
1. Validate Multikey.
2. Enforce multicodec `ML-DSA-44`.
3. Decode to raw public key.
4. Verify DID signature using decoded raw public key.
5. Derive DID from raw key and genesis.
6. Store initial DID record with one Authentication key (`#update`).

## On-Chain Mutation Functions

- `create_did(public_key, did_signature)`
- `add_key(did_id, key_id_suffix, vm_type, public_key, roles, controller, did_signature)`
- `revoke_key(did_id, key_id, did_signature)`
- `deactivate_did(did_id, did_signature)`
- `add_service(did_id, service, did_signature)`
- `remove_service(did_id, service_id, did_signature)`
- `set_metadata(did_id, entry, did_signature)`
- `remove_metadata(did_id, key, did_signature)`
- `rotate_key(did_id, old_key_id, new_public_key, new_key_id_suffix, new_vm_type, new_controller, roles, did_signature)`
- `update_roles(did_id, key_id, roles, did_signature)`

## Validation Rules

- DID signatures are checked against the single active Authentication key.
- DID signatures for all mutation calls MUST be verified against key id `#update`.
- `controller` (if present) must be a strict DID URI (`did:qsb:<did-id>` format).
- `service.id`:
  - fragment form `#...` (validated), or
  - absolute URI.
- `service.serviceEndpoint`: absolute URI.
- key selection for `revoke_key`, `rotate_key` (old key), and `update_roles` is by `key_id` (DID URL), not by raw public key.
- `revoke_key` MUST reject revocation of key id `#update`.
- `rotate_key` on key id `#update` MUST preserve key id `#update` for the new key material.
- `key_id_suffix`:
  - user provides suffix like `key-1` or `#key-1`,
  - runtime normalizes to `#...`,
  - rejects full DID values,
  - enforces uniqueness.

## Deactivation and Revocation

- Deactivation is irreversible.
- After deactivation, mutations are rejected.
- Revoked keys remain in state and are excluded from active DID Document sections.

## DID Resolution Output

Resolver returns DID Resolution result:
- `didDocument`
- `didDocumentMetadata`
- `didResolutionMetadata`

Current `didResolutionMetadata.contentType`:
- `application/did+ld+json`

Current DID Document mapping includes:
- `@context`
- `id`
- `verificationMethod`
- `authentication`
- `assertionMethod`
- `keyAgreement`
- `capabilityInvocation`
- `capabilityDelegation`
- `service`

### DID Resolution Errors

Resolver behavior MUST follow:
- malformed DID input -> `didResolutionMetadata.error = "invalidDid"`, `didDocument = null`
- syntactically valid but absent DID -> `didResolutionMetadata.error = "notFound"`, `didDocument = null`
- successful resolution -> `didResolutionMetadata.error = null`, `didDocument` populated

Implementations SHOULD preserve this behavior in the off-chain resolver API.

## Runtime API and RPC

Runtime API:
- `did_by_string(did: Vec<u8>) -> Option<DidDetails>`

Node RPC:
- `did_getByString`

## Off-Chain Resolver

Reference resolver (`qsb-did-resolver`) provides:
- HTTP endpoint: `GET /1.0/identifiers/{did}`
- DID syntax guard for `did:qsb:...`
- mapping from `did_getByString` result to DID Resolution result.

Runtime and node RPC expose raw DID state (`DidDetails`) only.
Canonical DID Resolution output is generated by the off-chain resolver.

Resolver interoperability requirements:
- `contentType` MUST be `application/did+ld+json` on successful resolution.
- `verificationMethod[].id` MUST be a DID URL.
- `verificationMethod[].controller` MUST be a DID URI.
- `verificationMethod[].type` MUST match runtime `vm_type` mapping (`Multikey` or `JsonWebKey2020`).

## Security and Privacy

- On-chain data are public and permanent.
- Private keys never go on-chain.
- Service endpoints / metadata can leak sensitive information.
- Compromised keys should be rotated or revoked immediately.
