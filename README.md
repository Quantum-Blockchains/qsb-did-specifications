# QSB DID — `did:qsb` Method Specification

## Description

`did:qsb` is a native on-chain DID method for the QSB blockchain (Substrate-based).
Chain runtime stores canonical DID state (`DidDetails`), while DID Resolution output is built by the off-chain resolver.

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" in this
document are to be interpreted as described in RFC 2119 and RFC 8174.

## DID Syntax

`did:qsb:<did-id>`

Where `did-id` is Base58 of exactly 32 bytes.

Runtime and resolver interfaces that accept DID identifiers MUST require full `did:qsb:<did-id>` syntax.

## DID Identifier Derivation

DID is derived from the decoded raw public key of the creation key:

```
material = "QSB_DID" || genesis_hash_32_bytes || raw_public_key
did_id_bytes = blake2_256(material)
did-id = base58(did_id_bytes)
did = "did:qsb:" || did-id
```

Notes:
- `blake2_256` is Substrate `sp_io::hashing::blake2_256`.
- `genesis_hash_32_bytes` provides chain separation.
- `create_did` enforces ML-DSA-44 codec for the creation/update key.

## Key Material Format

`did:qsb` runtime is **Multikey-only**.

Current runtime behavior:
- `create_did` requires Multikey and enforces codec `0x1210` (ML-DSA-44).
- `add_key` and `rotate_key` require Multikey, validate encoding, decode codec + raw key, and store normalized values.

### Multikey Validation

Runtime validates:
- multibase prefix `u`,
- Base64URL (no padding) decodability,
- leading unsigned varint multicodec,
- non-empty key bytes after codec.

Policy:
- `create_did` MUST reject codecs other than `0x1210`.
- `add_key` and `rotate_key` accept any syntactically valid Multikey codec/value.

## On-Chain DID State (`DidDetails`)

Per DID:
- `version: u64`
- `deactivated: bool`
- `keys: Vec<DidKey>`
- `services: Vec<ServiceEndpoint>`
- `metadata: Vec<MetadataEntry>`
- `next_key_index: u32`

`DidKey`:
- `key_id` (full DID URL, e.g. `did:qsb:...#update`)
- `multicodec: Option<u64>`
- `public_key` (normalized raw bytes)
- `roles`
- `controller` (optional strict DID URI)
- `revoked`

## Key Roles

- `Authentication`
- `AssertionMethod`
- `KeyAgreement`
- `CapabilityInvocation`
- `CapabilityDelegation`

## Authorization Model

DID mutation authorization is bound to a fixed update key id:

- `did:qsb:<did-id>#update`

Rules:
- All mutation-call DID signatures MUST verify against the non-revoked `#update` key.
- Additional keys MAY also carry `Authentication`, but they do not authorize state mutation unless they are the `#update` key.
- `revoke_key` MUST reject revoking `#update`.
- `rotate_key` for `#update` MUST preserve `#update` as key id on the new key.

## DID Creation

Wallet flow:
1. Generate ML-DSA-44 keypair.
2. Build Multikey for public key.
3. Build payload: `"QSB_DID_CREATE" || SCALE(multikey_bytes)`.
4. Sign payload with ML-DSA-44 private key.
5. Submit `create_did(multikey_bytes, did_signature)`.

Runtime flow:
1. Validate Multikey.
2. Enforce codec `0x1210` (ML-DSA-44).
3. Decode raw public key.
4. Verify DID signature with decoded raw public key.
5. Derive DID from raw key + genesis.
6. Store initial key as `key_id = did:qsb:<did-id>#update`.

## On-Chain Mutation Functions

- `create_did(public_key, did_signature)`
- `add_key(did_id, key_id_suffix, public_key, roles, controller, did_signature)`
- `revoke_key(did_id, key_id, did_signature)`
- `deactivate_did(did_id, did_signature)`
- `add_service(did_id, service, did_signature)`
- `remove_service(did_id, service_id, did_signature)`
- `set_metadata(did_id, entry, did_signature)`
- `remove_metadata(did_id, key, did_signature)`
- `rotate_key(did_id, old_key_id, new_public_key, new_key_id_suffix, new_controller, roles, did_signature)`
- `update_roles(did_id, key_id, roles, did_signature)`

## Validation Rules

- DID signatures for mutation calls MUST verify against `#update`.
- `controller` (if present) must be a strict DID URI (`did:qsb:<did-id>`).
- `service.id`:
  - fragment form `#...` (validated), or
  - absolute URI.
- `service.serviceEndpoint`: absolute URI.
- key selection for `revoke_key`, `rotate_key` (old key), and `update_roles` is by `key_id` (DID URL).
- `key_id_suffix`:
  - input accepted as `key-1` or `#key-1`,
  - runtime normalizes to `#...`,
  - full DID values are rejected,
  - uniqueness is enforced.

## Deactivation and Revocation

- Deactivation is irreversible.
- After deactivation, mutations are rejected.
- Revoked keys remain in state and are excluded from active DID Document verification relationships.

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

Resolver behavior:
- malformed DID input -> `didResolutionMetadata.error = "invalidDid"`, `didDocument = null`
- syntactically valid but absent DID -> `didResolutionMetadata.error = "notFound"`, `didDocument = null`
- successful resolution -> `didResolutionMetadata.error = null`, `didDocument` populated

## Runtime API and RPC

Runtime API:
- `did_by_string(did: Vec<u8>) -> Option<DidDetails>`

Node RPC:
- `did_getByString`

## Off-Chain Resolver

Reference resolver (`qsb-did-resolver`) provides:
- HTTP endpoint: `GET /1.0/identifiers/{did}`
- DID syntax validation for `did:qsb:...`
- mapping from `did_getByString` raw state to DID Resolution response.

Runtime/node expose raw state only. Canonical DID Resolution output is resolver responsibility.

Resolver interoperability requirements:
- `contentType` MUST be `application/did+ld+json` on success.
- `verificationMethod[].id` MUST be a DID URL.
- `verificationMethod[].controller` MUST be a DID URI.
- `verificationMethod[].type` MUST be `Multikey`.

## Security and Privacy

- On-chain DID state is public and permanent.
- Private keys MUST remain off-chain.
- Service endpoints and metadata may leak sensitive data.
- Compromised update key should be rotated immediately.
