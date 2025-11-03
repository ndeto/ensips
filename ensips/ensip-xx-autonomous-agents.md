---
title: ENS Autonomous Agents
author: Prem Makeig (premm.eth) <premm@unruggable.com>, Martin Ndeto (ndeto.eth) <martin@unruggable.com>
discussions-to: <URL>
status: Draft
created: 2025-11-03
---


# ENS Autonomous Agents

## Abstract  
Defines standardized ENS records that enable ENS names to represent autonomous entities capable of acting, interacting, and authenticating across execution environments. The `agent-registry:<chain-identifier>` record provides a verifiable binding to onchain registries, while the `agent:context:*` namespace defines structured metadata describing an entity’s capabilities and operational parameters. The `<chain-identifier>` used in the record complies with **[ERC-7930](https://eips.ethereum.org/EIPS/eip-7930)** chain identifers.

## Motivation  
As autonomous agents proliferate across multiple networks and execution environments, the need for verifiable identifiers that maintain consistent identity and context becomes critical. ENS provides a universal naming layer linking human-readable identifiers to verifiable onchain data, making it a natural foundation for autonomous entities that operate across multiple chains.

## Specification

The key words “MUST”, “SHOULD”, and “MAY” in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### 1. Agent Registry ENS Record  
The `agent-registry:<chain-identifier>` record defines the authoritative link between an ENS name and its registered identity within an **[ERC‑8004](https://eips.ethereum.org/EIPS/eip-8004)** agent registry. It allows resolvers and clients to confirm that a given ENS name corresponds to a valid agent entry on a specific chain. 

Each record encodes verification data in a compact binary layout: a 20‑byte `registryAddress` (EVM address), followed by a 1‑byte `agentIdLength`, and finally the `agentId` bytes of variable length. This tuple is hex‑encoded with a `0x` prefix in the ENS record.

```
0x<registryAddress><agentIdLength><agentId>
```

Clients MUST interpret the `agent-registry:<chain-identifier>` record according to this encoding. Verification between ENS and an agent registry is achieved by using these records to establish bidirectional trust of the association between an ENS name and its registry-backed identity.

### Verification Methods

Starting from an agent registry:
1. The client MUST query the ENS name of the agent using the metadata exposed by the registry.  
2. The client MUST determine the corresponding ERC‑7930 chain identifier on which the registry is deployed.  
3. The client MUST forward resolve the `agent-registry:<chain-identifier>` record for the specified ENS name.  
4. The client MUST parse the hex‑encoded value to extract the registry address, `agentIdLength`, and `agentId`.  
5. The client MUST verify that the registry address corresponds to the registry contract implementing the expected interface.  
6. The client MUST verify that the extracted `agentId` corresponds to the expected agent entry within that registry.  

Starting from an ENS name:
1. The client MUST determine the target ERC‑7930 chain identifier for the agent registry.  
2. The client MUST query the `agent-registry:<chain-identifier>` record for that ENS name.  
3. The client MUST parse the hex‑encoded value to extract the registry address, `agentIdLength`, and `agentId`.  
4. The client MUST verify that the registry contract decoded from the ENS record is the expected agent registry on the specified chain.
5. The client MUST verify that the registry entry for the agent explicitly references the ENS name being resolved.  

These verification methods ensure bidirectional consistency between onchain registry entries and ENS-resolved identities, forming a verifiable trust root across execution environments.

Registries implementing this ENSIP SHOULD conform to established standards such as ERC‑8004 for agent registry design. The specific registry standard used MUST be publicly documented and accessible to clients for proper verification.

### Example Multichain Agent

An ENS name can reference multiple agent registries across chains using the ERC‑7930 format:

agent-registry:000100000100 → **Ethereum Mainnet** (EIP‑155 chain ID 1) registry  
agent-registry:00010000018900 → **Polygon** (EIP‑155 chain ID 137) registry  
agent-registry:00010002A86A00 → **Avalanche** (EIP‑155 chain ID 43114) registry  

Each record encodes the ERC‑7930 chain identifier, registry address, and agentId for that chain.

### 3. Agent Context

The `agent:context:*` namespace defines a standardized interface for storing contextual data associated with an ENS name. This allows autonomous agents to expose metadata defining how they interact with other agents, contracts, or services. Each record key under this namespace provides semantically scoped information such as capabilities, preferences, language, supported interfaces, or operational parameters.

Each key follows the format:

```
agent:context:<attribute>
```

where `<attribute>` identifies a specific category of context. Values MAY be UTF-8 encoded text when stored via `text()` resolvers or arbitrary binary data when stored via `data()` resolvers conforming to ENSIP‑24. 

Clients resolving these records SHOULD interpret the available keys as interaction parameters rather than strict schemas. The namespace is intentionally flexible, allowing decentralized definition of new attributes.

#### 3.1 Context Record Semantics

Implementers MAY publish a schema reference under `agent:context:schema` specifying the expected keys and formats for that agent’s context. This provides a simple, composable mechanism for defining hierarchical relationships between agents without prescribing specific delegation semantics.

All `agent:context:*` records are public metadata; sensitive or operational secrets MUST NOT be published. The namespace supports flexible serialization formats, and JSON is RECOMMENDED for interoperability.

Example (non-normative):

```
agent:context:persona             = "operator"
agent:context:capability          = ["swap", "oracle"]
agent:context:locale              = "en-US"
agent:context:payment:preferred   = "x402:ETH:0xabc123..."
agent:context:collection          = "unruggable.eth"
agent:context:schema              = "ipfs://bafy.../schema.json"
agent:context:parent              = "organization.eth"
agent:context:delegate            = "wallet.unruggable.eth"
```

### Data Resolver Compatibility

Implementations of this ENSIP MAY use resolvers supporting **[ENSIP-5: Text Records](https://docs.ens.domains/ensip/5)** or **[ENSIP-24: Arbitrary Data Resolution](https://docs.ens.domains/ensip/24)** to store and retrieve agent-related records, with `data(bytes32 node, string key)` enabling compact binary encoding of contextual and registry data while maintaining compatibility with existing ENS interfaces. Clients resolving agent records SHOULD support both `text()` and `data()` resolver interfaces to ensure forward compatibility and allow efficient representation of structured agent metadata. Implementers SHOULD prefer `data()` for binary or structured payloads and `text()` for small human-readable values.

## Conclusion

This ENSIP defines the foundation for verifiable cross-chain identity for autonomous entities. By combining registry-based verification and contextual metadata, ENS becomes a universal layer for agentic interoperability across execution environments.

## References

- **[ENSIP-5: Text Records](https://docs.ens.domains/ensip/5)** — Defines the text record resolver profile for storing UTF-8 encoded text data on ENS names.
- **[ENSIP-24: Arbitrary Data Resolution](https://docs.ens.domains/ensip/24)** — Introduces a resolver profile for arbitrary bytes-based data resolution.
- **[ERC-8004: Agent Registry Standard](https://eips.ethereum.org/EIPS/eip-8004)** — Defines the canonical onchain registry interface for agent registration and verification.
- **[ERC‑7930: Interoperable Chain and Address Identifier Standard](https://eips.ethereum.org/EIPS/eip-7930)** — Defines standardized byte encoding for chain identifiers used across registries.
- **[RFC 2119: Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)** — Defines the interpretation of MUST, SHOULD, and MAY.