---
title: Coin Type field
description: 	Introduces a resolver field that returns the canonical coin type for a blockchain using its ENS name.
author: Martin Ndeto (ndeto.eth)
discussions-to: <URL>
status: Idea
created: 2025-10-15
requires: ENSIP-10, ENSIP-11
---

## Abstract

This ENSIP introduces a new resolver field, `coinType(bytes32 node)`, which returns the canonical coin type for a blockchain using its ENS name. This provides a standardized, protocol-level primitive that enables other specifications -  such as [ERC-7828](https://eips.ethereum.org/EIPS/eip-7828) (which implements [ENSIP-9](https://docs.ens.domains/ensip/9)) - to unambiguously resolve cross-chain addresses (for example, `vitalik.eth@optimism`).

## Motivation

Cross-chain resolution flows often require access to a blockchain’s canonical coin type identifier. Existing approaches depend on heterogeneous sources such as [SLIP-44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) mappings or derivations from the EVM chain ID (as defined in [ENSIP-11](https://docs.ens.domains/ensip/11)), introducing inconsistency and additional parsing overhead.

Under the ERC-7828 specification, a chain identifier (for example, `optimism`) is resolved through the ENS Registry using a domain such as `optimism.on.eth`. Because the registry does not natively store EVM chain IDs, consumers must currently query auxiliary text or data records - or rely on external mappings - to determine the correct coin type.

The introduction of a dedicated resolver field, `coinType(bytes32 node)`, provides a canonical, lightweight, and unambiguous method for retrieving the coin type associated with an ENS name that represents a blockchain. This establishes a standardized, protocol-level primitive for ERC-7828 implementers, wallets, and interoperability tooling to resolve cross-chain addresses reliably and efficiently.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Resolver Interface

Resolvers that implement this ENSIP **MUST** support the following interface:

```solidity
function coinType(bytes32 node) external view returns (uint256);
```

The `coinType(bytes32 node)` function **MUST** return the canonical coin type associated with the ENS name represented by `node`.  
The returned value **MUST** conform to the SLIP-44 registry, the coin type derivation defined by ENSIP-11, or an equivalent canonical mapping defined by the represented blockchain.

If the ENS name does not represent a registered blockchain, or if no coin type is defined, the function **MUST** revert.

### Integration

This field integrates directly into the existing resolver profile model alongside `addr` (ENSIP-9), `contenthash` ([ENSIP-7](https://docs.ens.domains/ensip/7)), and `ABI` ([ENSIP-4](https://docs.ens.domains/ensip/4)), enabling gas-efficient onchain lookups.  
Implementers of ERC-7828 (which implements ENSIP-9 multichain address resolution) **MAY** call this function to determine the coin type for a given blockchain domain (e.g., `optimism.on.eth`) before resolving cross-chain addresses such as `vitalik.eth@optimism`.  

## ERC-165 Interface Detection

Resolvers implementing this ENSIP **MUST** implement [ERC-165](https://eips.ethereum.org/EIPS/eip-165) and **MUST** return `true` for the following interface identifiers:

- **ERC-165:** `0x01ffc9a7`
- **Extended Resolver ([ENSIP-10](https://docs.ens.domains/ensip/10)):** `0x9061b923` (if implemented)
- **CoinType Resolver (this ENSIP):** `type(ICoinTypeResolver).interfaceId`

```solidity
interface ICoinTypeResolver {
    function coinType(bytes32 node) external view returns (uint256);
}
```

## Extended Resolver (ENSIP-10)

Callers **MAY** read CoinType via the extended resolver entrypoint by encoding the `coinType` selector:

```solidity
// name: DNS-encoded ENS name representing a chain, e.g., "optimism.on.eth"
// node: namehash(name)
resolve(name, abi.encodeWithSelector(ICoinTypeResolver.coinType.selector, node));
// Returns: ABI-encoded uint256 in the resolve() return bytes
```

## Semantics

- **Scope:** `coinType(bytes32)` is specific to resolvers implementing the Chain Registry profile. Other ENS resolvers that do not support this interface (e.g., standard name resolvers) will return `false` for `supportsInterface(0x28be03e4)` and are therefore outside the scope of this ENSIP.
- **Canonical value:**
  - EVM chains **MUST** use the ENSIP-11 convention: `coinType = 0x80000000 | chainId`.
  - Non-EVM chains **MUST** use their canonical SLIP-44 coin type, when one exists.
  - **Unset values:** Because `0` is a valid coin type (Bitcoin), implementations MUST NOT use `0` as a sentinel for unset. Implementations SHOULD revert with a custom error (e.g., `CoinTypeNotSet(bytes32 node)`).

## Usage with ERC-7828

Given an identifier like `vitalik.eth@optimism`:
1. Query `optimism.on.eth` on the `on.eth` registry to obtain the chain resolver that supports `coinType(bytes32)`.
2. Compute `node = labelhash("optimism.on.eth")`
3. Fetch the chain’s coin type: `resolver.coinType(node) → returns uint256`.
4. Use that coin type to resolve the multichain address for the ENS name: `resolve.addr(namehash("vitalik.eth"), coinType)`, per ENSIP-9.
5. The result is the address of vitalik.eth on the Optimism chain.

## Rationale

The primary rationale for this ENSIP is to provide a primitive, protocol-level mechanism for accessing a blockchain’s coin type directly through its ENS name. This enables cross-chain address resolution, identity mapping, and interoperability without relying on external mappings, text records, or derived data.

## Backwards Compatibility

Implementations **MAY** optionally mirror the value to a textual compatibility key (e.g., `text(node, "coin-type")`) or to a generic data record (e.g., `data(node, "coin-type")`) for discoverability. Such mirrors are non-normative and outside this ENSIP.

## Security Considerations

- **Correctness:** Incorrect coin types can misdirect address resolution. Registries **SHOULD** implement appropriate governance and review processes for updates.
- **Trust boundaries:** Clients **SHOULD** only consume values from registries or names they explicitly trust.
- **Unset handling:** Because `0` is valid for Bitcoin, implementations **MUST** revert when unset rather than returning `0`.

## Reference Interface (Solidity)

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.19;

interface ICoinTypeResolver /* is IERC165 */ {
    function coinType(bytes32 node) external view returns (uint256);
}

// Example setter
function setCoinType(bytes32 node, uint256 coinType) external;
// Implementers SHOULD return true for type(ICoinTypeResolver).interfaceId in supportsInterface.
```

## References

- [ENSIP-10](https://docs.ens.domains/ensip/10) — Extended Resolver (resolve)  
- [ENSIP-11](https://docs.ens.domains/ensip/11) — EVM compatible Chain Address Resolution (coin type derivation)  
- [ENSIP-9](https://docs.ens.domains/ensip/9) — Cross-chain address resolution via ERC-7828  
- [ENSIP-7](https://docs.ens.domains/ensip/7) — Contenthash field (record precedent)  
- [ENSIP-4](https://docs.ens.domains/ensip/4) — Support for contract ABIs  
- [ERC-165](https://eips.ethereum.org/EIPS/eip-165) — Standard Interface Detection  
- [ERC-7828](https://eips.ethereum.org/EIPS/eip-7828) — Cross-chain address resolution  
- [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) — Registered coin types  

## Copyright

Copyright and related rights waived via  
[CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
