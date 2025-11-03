---
title: ENS Autonomous Agents
author: Prem Makeig (premm.eth) <premm@unruggable.com>, Martin Ndeto (ndeto.eth) <martin@unruggable.com>
discussions-to: <URL>
status: Draft
created: 2025-11-03
---


# ENS Autonomous Agents

## Abstract  
Defines standardized ENS records that enable ENS names to represent autonomous entities that can act, interact, and authenticate across execution environments. This specification defines two complementary mechanisms that together establish verifiable identity and contextual data for autonomous agents. For the purposes of this specification, the term “agent” refers to an autonomous entity represented by an ENS name. 

The `agent-registry:<chain-id>` record establishes a verifiable binding between an ENS name and its registry-backed onchain identity, ensuring authenticity across execution environments. By parameterizing the record with a chain identifier, the system remains chain-agnostic, thereby enabling precise cross-chain verification. 

The `agent:context:*` namespace defines a flexible structure for representing contextual metadata that describes an agent’s capabilities, preferences, and operational parameters. 

## Motivation  
As autonomous agents proliferate across multiple networks and execution environments, the need for verifiable identifiers that maintain consistent identity and context becomes critical. ENS is well positioned to fulfill this role by providing a universal naming layer linking human-readable names to verifiable onchain data.

Existing ENS records do not fully address requirements such as cross-chain authenticity, structured context, and delegation semantics. This standard introduces a unified framework that allows ENS names to function as verifiable, interoperable identities across systems and chains.

## Specification

> The key words “MUST”, “SHOULD”, and “MAY” in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### 1. Agent Registry ENS Record  
The `agent-registry:<chain-id>` text record defines the authoritative link between an ENS name and its registered identity within an **[ERC‑8004: Agent Registry Standard](https://eips.ethereum.org/EIPS/eip-8004)** agent registry. It allows resolvers and clients to confirm that a given ENS name corresponds to a valid agent entry on a specific chain. 

Each record encodes verification data in a compact binary layout: a 20‑byte `registryAddress` (EVM address), followed by a 1‑byte `agentIdLength`, and finally the `agentId` bytes of variable length. This tuple is hex‑encoded with a `0x` prefix in the ENS text record.

```
0x<registryAddress><agentIdLength><agentId>
```

Clients MUST interpret the `agent-registry:<chain-id>` record according to this encoding and perform bidirectional verification as specified below.

### Verification Methods

Verification between ENS and an agent registry is achieved by the use of the `agent-registry:<chain-id>` ENS text record, which encodes the registry address, `agentIdLength`, and `agentId` in a hex format. This enables bidirectional verification of the association between an ENS name and its registry-backed identity.

#### 1. Registry-to-ENS Verification
When starting from an agent registry:
1. The client MUST query the ENS name of the agent using the metadata exposed by the registry.  
2. The client MUST determine the corresponding chain ID on which the registry is deployed.  
3. The client MUST forward resolve the ENS text record `agent-registry:<chain-id>` for the specified ENS name.  
4. The client MUST parse the hex‑encoded value to extract the registry address, `agentIdLength`, and `agentId`.  
5. The client MUST verify that the registry address corresponds to the registry contract implementing the expected interface.  
6. The client MUST verify that the extracted `agentId` corresponds to the expected agent entry within that registry.  

#### 2. ENS-to-Registry Verification
When starting from an ENS name:
1. The client MUST determine the target chain ID for the agent registry.  
2. The client MUST query the ENS text record `agent-registry:<chain-id>` for that ENS name.  
3. The client MUST parse the hex‑encoded value to extract the registry address, `agentIdLength`, and `agentId`.  
4. The client MUST verify that the registry contract decoded from the ENS record is the expected agent registry on the specified chain.
5. The client MUST verify that the registry entry for the agent explicitly references the ENS name being resolved.  

Registries implementing this ENSIP SHOULD conform to established standards such as ERC‑8004 for agent registry design. The specific registry standard used MUST be publicly documented and accessible to clients for proper verification.

### 3. Agent Context

The `agent:context:*` namespace defines a standardized interface for storing contextual data associated with an ENS name. This allows autonomous agents to describe metadata defining how they interact with other agents, contracts, or services. Each record key under this namespace provides semantically scoped information such as capabilities, preferences, language, supported interfaces, or operational parameters.

Each key follows the format:

```
agent:context:<attribute>
```

where `<attribute>` identifies a specific category of context. Values MAY be UTF-8 encoded text when stored via `text()` resolvers or arbitrary binary data when stored via `data()` resolvers conforming to ENSIP‑24. This flexibility allows context records to represent structured formats such as JSON, YAML, or other serializations appropriate to the resolver profile.

Clients resolving these records SHOULD interpret the available keys as interaction parameters rather than strict schemas. The namespace is intentionally flexible, allowing decentralized definition of new attributes.

#### 3.1 Context Record Semantics

Implementers MAY publish a schema reference under `agent:context:schema` specifying the expected keys and formats for that agent’s context. Agents MAY inherit context from another ENS name by referencing it via `agent:parent:<ENS>`. Systems interpreting context records SHOULD resolve and merge inherited context hierarchically, applying overrides from the child record where conflicts occur.

Delegation relationships can be modeled through paired records (`agent:context:delegate:<ENS>` and `agent:context:parent:<ENS>`), which, when set reciprocally, define a valid delegation link between agents.

All `agent:context:*` records are public metadata; sensitive or operational secrets MUST NOT be published. The namespace supports flexible serialization formats, and JSON is RECOMMENDED for interoperability.

Example (non-normative):

```
agent:context:persona             = "operator"
agent:context:capability          = ["swap", "oracle"]
agent:context:locale              = "en-US"
agent:context:payment:preferred   = "x402:ETH:0xabc123..."
agent:collection                  = "unruggable.eth"
agent:context:schema              = "ipfs://bafy.../schema.json"
agent:context:schema              = "ipfs://bafy.../schema.json"
agent:parent                      = "organization.eth"
```

### Data Resolver Compatibility

Implementations of this ENSIP MAY use resolvers supporting **[ENSIP-5: Text Records](https://docs.ens.domains/ensip/5)** or **[ENSIP-24: Arbitrary Data Resolution](https://docs.ens.domains/ensip/24)** to store and retrieve agent-related records. Using `data(bytes32 node, string key)` enables compact or binary encoding of contextual and registry data while maintaining compatibility with existing ENS interfaces.  
Clients resolving agent records SHOULD support both `text()` and `data()` resolver interfaces to ensure forward compatibility and allow efficient representation of structured agent metadata.  
Implementers SHOULD prefer `data()` for binary or structured payloads and `text()` for small human-readable values.

## Conclusion

This ENSIP defines a unified framework that enables ENS names to represent autonomous agents as verifiable, interoperable identities across chains. By combining registry-based verification with flexible contextual metadata, this standard establishes a foundation for verifiable, cross-chain identity.  
Future ENSIPs may extend it with formal schemas or registry enhancements, but this specification defines the core model for agentic interoperability.

## References

- **[ENSIP-5: Text Records](https://docs.ens.domains/ensip/5)** — Defines the text record resolver profile for storing UTF-8 encoded text data on ENS names.
- **[ENSIP-24: Arbitrary Data Resolution](https://docs.ens.domains/ensip/24)** — Introduces a resolver profile for arbitrary bytes-based data resolution.
- **[ERC-8004: Agent Registry Standard](https://eips.ethereum.org/EIPS/eip-8004)** — Defines the canonical onchain registry interface for agent registration and verification.
- **[RFC 2119: Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)** — Defines the interpretation of MUST, SHOULD, and MAY.