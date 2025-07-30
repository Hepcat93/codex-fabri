# ⚡ Gas Optimization Features in Solidity

This document summarizes Solidity gas optimization techniques based on hands-on examples and explanations cherry-picked by me dyring my studies. It covers struct packing, custom errors, and efficient storage access — with practical illustrations and gas comparisons.

---

## 📌 Structs vs Individual Variables

> **Why?** Efficient storage layout = fewer storage slots = cheaper gas.

### 🧠 EVM Storage Basics

* Each storage slot = **32 bytes** (256 bits)
* Solidity tries to **pack variables top to bottom** into slots
* If a variable doesn’t fit → next slot used (wasteful)

---

### ❌ Inefficient Layout Example

```solidity
contract Example1 {
    bool a;        // slot 0
    uint256 b;     // slot 1
    bool c;        // slot 2 → wastes space!
}
```

* `bool` occupies only 1 byte but wastes 31 in each slot.

---

### ✅ Optimized Layout with Struct

```solidity
contract Example2 {
    struct Packed {
        bool a;
        bool c;
        uint256 b;
    }

    Packed data; // `a` and `c` can be packed into 1 slot
}
```

Here, `a` and `c` can share the same slot (bit-level), saving gas.

---

#### :wrench: In practice

* Packing **only helps in `storage`**, not in `memory` or `stack`.
* Best used when you have **many small variables** and gas cost matters.
* The order **matters**: place smaller types first in structs to help the compiler pack them efficiently.

---

#### :no_entry_sign: Caveat

Sometimes structs can introduce **slightly more complexity** (e.g., accessing fields), so use them when:

* Storage efficiency matters (e.g., many records in a mapping)
* You're optimizing for gas
* The struct groups logically related data

---

## ❗ Custom Errors vs `require` Statements

> **Why?** Saves gas by avoiding long error strings.

### ✅ Custom Error Example

```solidity
error FundMe__NotOwner();

modifier onlyOwner() {
    if (msg.sender != i_owner) {
        revert FundMe__NotOwner();
    }
    _;
}
```

### ❌ Traditional `require`

```solidity
require(msg.sender == owner, "Not owner");
```

### 🔎 Comparison

| Feature       | `require(...)` with message | `revert CustomError()` |
| ------------- | --------------------------- | ---------------------- |
| Gas usage     | High (stores full string)   | Low                    |
| Debug clarity | Medium                      | High                   |
| Type safety   | ❌                           | ✅ (with args support)  |

---

## 🔄 Storage Access in Loops vs Memory Copy

> **Why?** Repeated storage reads are expensive. Memory is cheaper.

### ❌ Costly: Direct Storage Loop

```solidity
for (uint256 i = 0; i < s_funders.length; i++) {
    address funder = s_funders[i];
}
```

### ✅ Efficient: Memory Copy First

```solidity
address[] memory funders = s_funders;
for (uint256 i = 0; i < funders.length; i++) {
    address funder = funders[i];
}
```
---

### :moneybag: 2. Why is `cheaperWithdraw()` Cheaper?

The key difference lies in **how the array `s_funders` is accessed**:

* In `withdraw()`, the loop reads directly from **storage** on every iteration (`s_funders.length`, `s_funders[index]`). Accessing storage is expensive in Solidity.
* In `cheaperWithdraw()`, the array is **copied to memory** once (`address[] memory funders = s_funders`), and the loop iterates over that in-memory copy — which is **much cheaper** than repeatedly accessing storage.

> Storage reads are costly. Memory reads are much cheaper.

---

### 💰 Gas Cost Comparison

| Function            | Operation                 | Approx Gas / Iteration | Total Gas (10 funders) |
| ------------------- | ------------------------- | ---------------------- | ---------------------- |
| `withdraw()`        | Storage reads in loop     | \~2100                 | \~25,000–30,000        |
| `cheaperWithdraw()` | Memory copy + memory loop | \~100 (after copy)     | \~7,000–10,000         |

> Storage reads: expensive (\~2100 gas)
> Memory reads: cheap (\~3 gas after initial copy)

---

## ✅ Over time I plan to expand this document with:

* `unchecked` arithmetic tricks
* `constant` and `immutable` usage
* Optimized mappings
* Gas reports via Foundry
