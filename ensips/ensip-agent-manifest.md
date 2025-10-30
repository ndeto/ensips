---
title: ENS Agent Manifest Records
author: Martin Ndeto (ndeto.eth)
discussions-to: https://github.com/ethereum/ensips/issues/<issue-number>
status: Draft
created: 2025-10-28
---

# ENSIP-TBD-XX: ENS Agent Manifest Records

## Abstract  
This ENSIP defines a standardized mechanism for ENS names associated with AI Agents (i.e, `agents.uniswap.eth`), to publish and verify **agent manifests**. These manifests are signed JSON directories that list collections of autonomous agents (e.g., AI agents, off-ramp agents, service nodes). 

Using two minimal text records ([ENSIP-5](https://github.com/ensdomains/ensips/blob/master/ensips/ensip-5.md)), `manifest:url` and `manifest:hash`, clients can resolve an ENS name to a verifiable manifest, fetch and verify it, and parse onboard endpoints (per [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) terminology) of agents. Optionally, collections may be anchored on-chain via [ERC-8041](https://github.com/ethereum/ERCs/pull/1237) `manifestRoot`, enabling higher assurance. Optional onchain anchoring via ERC 8041 can be referenced per collection for higher assurance.

## Motivation  
As autonomous agents multiply across decentralized systems, there is a growing need for **trust-minimized discovery and verification** of legitimate agent identities and capabilities. Existing standards such as ERC‑8004 (agent identity & reputation) and ERC‑8041 (agent collections) define on-chain membership, but do not specify a universal, mirror-friendly directory format. This ENSIP fills that gap by leveraging ENS names for discovery and a content-addressed manifest schema for secure off-chain indexing and parsing.

### Terminology
**Collection** means a group of agents discoverable under an ENS name  
**Agent** means an individual autonomous identity  
**Endpoint** refers to a capability exposed by an agent per ERC 8004.

## Specification  
The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY” and “OPTIONAL” in this document are interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### ENS Text Records  
For any ENS name participating under this spec, the following baseline text records **MUST** be present:

```

text("manifest:url")   – URL to the root manifest (HTTP(S), IPFS, Arweave, ENS, IPNS, etc.)
text("manifest:hash")  – Hash of the raw manifest bytes (e.g., sha256-<hex> or multihash)

```

Clients **MUST** fetch the manifest from `manifest:url`, compute the hash of the raw bytes, then compare to `manifest:hash`. If they do not match, clients **MUST** reject the manifest.

Endpoint-scoped helper keys **MAY** be published to provide a direct lookup for a single collection file:

```

text("endpoint:<Name>:url")
text("endpoint:<Name>:hash")

```

Resolvers implementing these helper keys SHOULD return empty bytes for unset values, matching ENS text record conventions. Clients **MAY** use endpoint-scoped keys for a direct fetch of a single collection file. When a lookup yields empty bytes or the keys are absent, clients **MUST** fall back to resolving the root manifest via `manifest:url` and `manifest:hash`.

**Optional** text record:

```

text("manifest:schema")          – URL or CID describing the manifest schema version

````

- `manifest:schema` lets publishers point clients to a canonical schema document, changelog, or semantic version so verifiers can ensure they understand new fields before parsing or enforcing additional metadata.


### Root Manifest Schema  
The root manifest is a signed JSON document with the following normative fields:

```json
{
  "schemaVersion": "1.0.0",
  "issuedAt": "<unix-timestamp>",
  "expiresAt": "<unix-timestamp>",
  "collections": [
    {
      "name": "<string>",
      "category": "<string>",
      "inlined": <boolean>,
      "membersUrl": "<string or null>",
      "membersHash": "<string or null>",
      "collection8041": {
        "chainId": "<number>",
        "address": "<0x...>",
        "snapshotBlock": "<number or null>"
      },
      "notes": "<string or null>",
      "meta": {}, // optional free-form metadata
    }
    ],
  "sources": [
    { "url": "<string>", "hash": "<string>" }
  ],
  "signer": "<address>",
  "signature": "<hex-encoded signature>"
}
````

#### Field semantics

* `schemaVersion` – version of this manifest format.
* `issuedAt`, `expiresAt` – timestamps controlling staleness.
* `collections[]` – list of constituent agent groups:

  * `name` – human label (e.g., “A2A”, “MCP”).
  * `category` – grouping label for the agents in this collection (distinct from ERC-8004 agent endpoints).
  * `inlined` – indicates whether agent entries are embedded directly within the root manifest.
  * `membersUrl` – URL of the file listing agents when entries are not inlined.
  * `membersHash` – hash of the `membersUrl` file; REQUIRED when `inlined=false`.
  * `collection8041` – OPTIONAL object giving onchain anchoring metadata:

    * `chainId` – chain where the ERC-8041 contract resides.
    * `address` – ERC-8041 contract address.
    * `snapshotBlock` – block height anchoring the manifest (nullable).

  * `notes` – OPTIONAL free-form string guidance for this collection.
  * `meta` – OPTIONAL metadata object (schema pointer, capability tags, etc.).
* `sources[]` – list of mirrors for the root manifest with `url` and `hash`.
* `signer` – address that signs the manifest.
* `signature` – ECDSA/ed25519 signature of the canonical JSON.

If `inlined=true`, both `membersUrl` and `membersHash` **MUST** be `null`. If `inlined=false`, both fields **MUST** be populated. When `collection8041` is present, clients **MAY** perform high-assurance checks against that ERC-8041 contract.

### Agent Members File Schema

When a collection uses `inlined=false`, the file at `membersUrl` is a signed JSON with:

```json
{
  "collection8041": { "chainId": <number>, "address": "<0x...>" },
  "category": "<string>",
  "schemaVersion": "1.0.0",
  "agentEntries": [
    {
      "id": "<string or tokenId>",
      "tokenId": "<string or null>", // prefer id == tokenId when 8041 exists
      "agentPubkey": "<address or public key>",
      "agentCard": "<string URL or CID>",
      "capsHash": "<string>"
    }
    // … more entries …
  ],
  "signer": "<address>",
  "signature": "<hex-encoded signature>"
}
```

Clients must verify `capsHash` when fetching each `agentCard` later. When `collection8041` metadata is present, clients should prefer `id == tokenId` and use the provided chain context for reconciliation.

### Verification Flow

1. Resolve the ENS name and read `manifest:url` and `manifest:hash`.
2. Fetch the root manifest and verify the bytes match `manifest:hash`.
3. Verify the manifest signature and check `issuedAt` and `expiresAt`.
4. Choose a collection by `category`:
   - If `inlined` is `true`, use the `agentEntries` embedded in the root manifest.
   - If `inlined` is `false`, fetch `membersUrl`, verify the bytes match `membersHash`, then verify the members file signature.
5. *(Optional high assurance)* If `collection8041` exists, read `manifestRoot` from that ERC-8041 contract on `chainId`, compare the fetched members file hash (or Merkle root) to `manifestRoot`.
6. *(Optional fast path)* If endpoint-scoped ENS keys exist for the chosen `category`, read `endpoint:<Name>:url` and `endpoint:<Name>:hash` and perform the members-file fetch, hash verification, and signature verification directly using that scoped file.

## Backwards Compatibility

This specification uses existing ENS record types and does not alter core ENS behavior. Clients unaware of these records continue functioning without disruption.

## Security Considerations

* `membersHash` **MUST** be present whenever `membersUrl` is provided.
* When `collection8041` metadata exists, clients should prefer `id == tokenId` or otherwise include both explicitly for each agent entry.
* Endpoint-scoped ENS keys are helpers only; clients **MUST** hash-verify and signature-verify the files they reference just as they do the root manifest.
* Signatures must tie to a known authority key that the manifest directs.
* `expiresAt` prevents safe-looking rollback attacks; clients should not use expired manifests.
* Optional 8041 anchoring improves anti-tamper guarantees but is not required for baseline.
* Large member files should be paginated or compressed to avoid DoS.

## Rationale

By leveraging ENS for discovery and a content-verified manifest structure, this ENSIP enables **portable, mirror-friendly, trust-minimized agent directories**. The optional ERC-8041 anchoring provides additional auditability without complicating the fast path.

## Implementation

* An ENS resolver implementing this profile (i.e., supporting `manifest:url`, `manifest:hash`) is recommended.
* SDKs and verification libraries should encapsulate the ENS lookup, manifest fetch, hash check, signature verification, and optional on-chain reconciliation flows.
* Publishers should link example manifest files through `manifest:schema` when available and share lightweight test fixtures so clients can exercise parsing, hashing, and signature checks end-to-end.

## References

- [ENSIP-5: Text Records](https://github.com/ensdomains/ensips/blob/master/ensips/ensip-5.md)
- [ERC-8004: Trustless Agents](https://eips.ethereum.org/EIPS/eip-8004)
- [ERC-8041: Fixed-Supply Agent NFT Collections](https://github.com/ethereum/ERCs/pull/1237)
- [RFC 2119: Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)
- [RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://www.rfc-editor.org/rfc/rfc8174)

## Appendix

**Example ENS records**

```
text("manifest:url")  = "ipfs://bafy.../root.json"
text("manifest:hash") = "sha256-0x1234..."
text("endpoint:A2A:url")  = "ipfs://bafy.../a2a.json"
text("endpoint:A2A:hash") = "sha256-0xaaaa..."
```

**Example Root Manifest (abbreviated)**

```json
{
  "schemaVersion": "1.1.0",
  "issuedAt": 1730080000,
  "expiresAt": 1732680000,
  "collections": [
    {
      "name": "A2A Core",
      "category": "A2A",
      "inlined": false,
      "membersUrl": "ipfs://bafy.../a2a.json",
      "membersHash": "sha256-0x7a3b...",
      "collection8041": {
        "chainId": 1,
        "address": "0xA2A8041...",
        "snapshotBlock": 21234567
      },
      "notes": "Primary A2A cohort",
      "meta": { "schema": "1.0.0" }
    }
  ],
  "sources": [
    { "url": "ipfs://bafy.../root.json", "hash": "sha256-0x1234..." }
  ],
  "signer": "0xRootSigner",
  "signature": "0x..."
}
```

**Example Members File (abbreviated)**

```json
{
  "collection8041": { "chainId": 1, "address": "0xA2A8041..." },
  "category": "A2A",
  "schemaVersion": "1.0.0",
  "agentEntries": [
    {
      "id": "42",
      "tokenId": "42",
      "agentPubkey": "0x04f9...",
      "agentCard": "ipfs://bafy.../42.json",
      "capsHash": "sha256-0xdeadbeef..."
    }
  ],
  "signer": "0xAuthorityKey",
  "signature": "0x..."
}
```

---
