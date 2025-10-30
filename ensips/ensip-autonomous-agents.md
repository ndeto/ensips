# ENSIP: Autonomous Agents in ENS

## Abstract
Defines ENS standards for representing autonomous entities (referred to as agents) that are verifiable, discoverable, and interoperable across execution environments.

## Motivation
The motivation behind this proposal is to extend ENS to support verifiable and programmable identity for autonomous agents and entities. These entities may include AI agents, protocol services, DAOs, validators, or other autonomous systems that need persistent and discoverable identity across execution environments.

Key drivers:
- **Trust and Authenticity:** Provide a verifiable root of trust linking each agent’s ENS name to its registry-backed onchain identity.
- **Discovery:** Enable clients to discover an entity’s capabilities and interfaces through standardized manifest and context records.
- **Consistency:** Ensure a single ENS name represents the same entity across execution environments, maintaining identity integrity and coherence.
- **Extensibility:** Support structured metadata via context and schema records, allowing agents to describe operational preferences and communication parameters.
- **Interoperability:** Introduce standardized payment and interaction endpoints (`agent:payment:*`) to enable automated, peer-to-peer value exchange between agents and services.

This proposal recognizes that autonomous agents increasingly require consistent identity surfaces to interact securely and transparently within decentralized ecosystems.

## Specification

The Agentic ENS standard is structured around four foundational pillars that define how entities and registries interact through ENS records.

### 1. Verification
Verification is performed natively across multiple chains using the `agent:registry:<chainId>` text record.

- **Core Record Pattern:** `agent:registry:<chainId>`
- **Mechanism:**  
  1. Each ENS name publishes one or more `agent:registry:<chainId>` text records, where `<chainId>` identifies the target network (e.g., `agent:registry:1`, `agent:registry:10`, `agent:registry:137`).  
  2. The record value is a hex‑encoded payload binding the ENS name to its verifiable registry address and entity identifier on that specific chain.  
  3. Verifiers perform both forward and reverse verification - resolving the ENS name to retrieve its registry record, then confirming that the onchain registry acknowledges the ENS name as its owner or entity.  
  4. The verified registry entries across chains form the trust anchors for manifest retrieval and context interpretation.

- **Purpose:**  
  Anchors authenticity to the onchain registry as the source of truth for each entity. The ENS record links to that registry entry, and verification confirms both directions - the ENS name points to the registry, and the registry acknowledges the ENS name. ENS provides naming, the registry provides authenticity, and manifests add interpretability.

- **Continuity:** multiple `agent:registry:<chainId>` records define presence across execution environments.

### 2. Discovery
Discovery defines how verifiers locate, verify, and interpret an entity’s metadata and capabilities from trusted sources once its registry authenticity has been established.

- **Core Records:**  
  `manifest:url`  
  `manifest:hash`

- **Mechanism:**  
  1. After verification, verifiers resolve the ENS name and fetch its manifest metadata via the `manifest:url` record.  
  2. The fetched data (usually JSON or equivalent structured format) is validated against the `manifest:hash` to confirm integrity.  
  3. Optionally, verifiers verify a signature inside the manifest linking it to the verified registry identity.  
  4. The manifest describes the entity’s capabilities, interfaces and callable capabilities, supported protocols, and declared collections or delegation scopes.  
  5. Verifiers can cache and mirror manifests off-chain using content-addressed storage gateways (e.g., IPFS) or other decentralized storage layers, referencing them via the same `manifest:hash`.

- **Purpose:**  
  Provides a structured, verifiable entry point for understanding an entity’s behavior and scope. It separates *what* the entity does (manifest) from *who* it is (registry verification), allowing entities to evolve their capabilities without breaking authenticity.

- **Trust Model:**  
  ENS anchors identity, the registry authenticates it, and the manifest interprets its functionality. Manifests must be immutable per hash and may reference optional `manifest:schema` or `manifest:sources` records for extended validation or indexing.


### 3. Context
Defines how entities express operational preferences, behavioral metadata, and interaction parameters.

- **Core Records:** `agent:context:*`
- **Optional Records:** `agent:payment:*`

- **Purpose:**  
  Allows entities to expose contextual and preferential data that guide interaction, authorization, and communication.  
  Payment records are conceptually linked to context but are defined separately for clarity and machine-level discoverability.  
  This separation enables transactional systems (e.g., wallets, relayers, payment channels) to query `agent:payment:*` directly, while coordination or reputation systems rely on `agent:context:*` for behavioral metadata and preferences.

- **Note:**  
  The `agent:payment:*` namespace may generalize in the future to `agent:interface:*` for broader interoperability across payment and other interface protocols.

- **Example Context Attributes:**  
  ```
  agent:context:persona       → defines role or behavioral archetype (e.g., researcher, broker)
  agent:context:capability    → high-level capability tags (e.g., swap, oracle, chat)
  agent:context:locale        → preferred human language or region (e.g., en-US, fr-FR)
  agent:context:format        → preferred data format (e.g., json, cbor)
  agent:context:protocols     → supported interface standards (e.g., erc-8004, x402)
  ```
  These examples provide a common vocabulary for consumers to interpret entity metadata consistently across systems.

- **Schema and Validation Model:**  
  Entities may publish an optional `agent:context:schema` key referencing a JSON schema or URI that defines valid context fields, e.g.:
  ```
  agent:context:schema = ipfs://bafy.../schema.json
  ```
  This enables decentralized standardization and validation of context fields without centralized control.

- **Inheritance and Hierarchy:**  
  Context records may optionally support hierarchical composition:
  - `agent:parent:<ENS>` — inherits base context from another ENS name (e.g., organization, registry, or collection).  
  - **Cross-environment overrides:** specify which keys override inherited context values.  
  This maintains modularity while allowing structured delegation and composition.

- **Security Note:**  
  Consumers should treat `agent:context:*` as **public metadata**.  
  Sensitive data or operational secrets must never be published through context or payment records.

### Cross-Environment Continuity (Core Consideration)

- ENS names represent a single entity identity consistently across execution environments.
- Verification, discovery, and context records work together to maintain persistent identity and capabilities.
- Supports seamless interoperability without redundant onchain or offchain identity mappings.
- Enables trust and discovery workflows that adapt to different execution environments naturally.


## Schema & Record Definition

Defines the canonical schema for Agentic ENS records and the expected key set for interoperability.

### Canonical Example (JSON)
```json
{
  "agent:registry:1": "0x1234...abcd",
  "agent:registry:10": "0xabcd...7890",
  "manifest:url": "ipfs://bafy.../root.json",
  "manifest:hash": "sha256-0x9f9c...",
  "agent:context:persona": "operator",
  "agent:context:capability": ["swap", "oracle"],
  "agent:context:locale": "en-US",
  "agent:payment:preferred": "ETH:0xabc123...",
  "agent:collection": "unruggable.eth"
}
```

### Record Specification

| Key | Description |
|------|--------------|
| `agent:registry:<chainId>` | Binds the ENS name to a verified registry entry on a given execution environment. |
| `manifest:url` | URI of the JSON manifest describing entity capabilities and metadata. |
| `manifest:hash` | SHA-256 or Keccak256 digest of the manifest file. |
| `agent:context:*` | Flat namespace for operational or behavioral metadata. |
| `agent:payment:*` | Defines payment or interaction interfaces. |
| `agent:collection` | Links the entity to a collection contract or ENS group context. (e.g ERC-8041)|
| `agent:context:schema` | Optional key referencing a JSON schema defining valid context fields. |


Schema records are optional but recommended for defining validation rules and ensuring consistent context interpretation across implementations.


### Manifest Schema (Example)

Defines the expected structure and validation rules for manifests referenced via `manifest:url` and `manifest:hash`.

#### Example Manifest Schema (JSON)
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Agentic ENS Manifest",
  "description": "Defines the structure and validation rules for an entity manifest referenced in ENS records.",
  "type": "object",
  "required": [
    "schemaVersion",
    "issuedAt",
    "entityId",
    "capabilities",
    "signer",
    "signature"
  ],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "description": "Version of the manifest schema used."
    },
    "issuedAt": {
      "type": "integer",
      "description": "Unix timestamp of when the manifest was generated."
    },
    "expiresAt": {
      "type": "integer",
      "description": "Optional expiration timestamp after which this manifest is invalid."
    },
    "entityId": {
      "type": "string",
      "description": "Unique identifier linking the manifest to the verified registry entity."
    },
    "capabilities": {
      "type": "array",
      "items": { "type": "string" },
      "description": "List of declared capabilities or supported actions."
    },
    "endpoints": {
      "type": "object",
      "description": "Optional interface endpoints or callable actions.",
      "additionalProperties": {
        "type": "object",
        "properties": {
          "uri": { "type": "string", "format": "uri" },
          "auth": { "type": "string" }
        },
        "required": ["uri"]
      }
    },
    "collections": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Optional list of collection ENS names or addresses this entity belongs to."
    },
    "signer": {
      "type": "string",
      "description": "Address or identifier of the signer who issued the manifest."
    },
    "signature": {
      "type": "string",
      "description": "Signature verifying the integrity and authenticity of the manifest content."
    }
  }
}
```

#### Example Manifest File (JSON)
```json
{
  "schemaVersion": "1.0.0",
  "issuedAt": 1730269800,
  "expiresAt": 1732875400,
  "entityId": "entity-12345",
  "capabilities": ["A2A", "OffRamp"],
  "endpoints": {
    "a2a": {"uri": "https://api.example.com/a2a", "auth": "mTLS"},
    "offramp": {"uri": "https://api.example.com/offramp", "auth": "sig-v1"}
  },
  "collections": ["unruggable.eth"],
  "signer": "0xSigner",
  "signature": "0x..."
}
```

Clients verifying a manifest should retrieve and validate it against this schema when the `manifest:schema` key is present in ENS records.


## ENSIPs to Merge

This unified Agentic ENS standard consolidates several related ENSIPs that collectively define entity identity, context, delegation, and multichain resolution. Each prior proposal contributes a key capability, now harmonized under a single specification.

| ENSIP | Title | Function | Integration |
|--------|--------|-----------|-------------|
| **ENSIP-TBD-11** | Multichain Name Resolution | Defines how ENS identities resolve consistently across multiple blockchains. | → Reframed as a **core cross-environment continuity consideration** applied across Verification, Discovery, and Context. |
| **ENSIP-TBD-14** | Agentic Systems Interface | Introduces the `root-context` text record to describe agentic systems and interfaces. | → Evolves into **`agent:context:*`** under the **Delegation & Context** pillar. |
| **ENSIP-TBD-15** | Agent Delegations | Defines reciprocal keys `agent-delegate:<ENS>` and `agent-parent:<ENS>` to establish parent ↔ delegate relationships. | → Extends the **Context** pillar with structured delegation and authorization. |
| **ENSIP-TBD-20** | AI Agent Registry ENS Name Verification | Defines `agent-registry` and the bidirectional ENS↔registry verification flow. | → Forms the foundation of the **Verification** pillar. |

These merged ENSIPs together create a comprehensive framework where ENS names act as verifiable, discoverable, and interoperable entity identities.