# Solidity Coding Conventions & Study Notes

This document collects structured, practical knowledge for smart contract auditing and development. It’s maintained as a study reference alongside the Discord server for fellow learners.

---

## 🔧 Full Contract Structure with Naming & Ordering Conventions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

// 1. Imports
import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {PriceConverter} from "./PriceConverter.sol";

// 2. Custom Errors
error MyContract__NotOwner();
error MyContract__InsufficientFunds();

// 3. Contract Declaration
contract MyContract {
    // 4. Type Declarations
    using PriceConverter for uint256;

    // 5. State Variables
    uint256 public constant MINIMUM_USD = 5 * 10 ** 18;
    address public immutable i_owner;
    AggregatorV3Interface private s_priceFeed;
    mapping(address => uint256) private s_addressToAmountFunded;
    address[] private s_funders;

    // 6. Events
    event Funded(address indexed funder, uint256 amount);
    event Withdrawn(address indexed owner, uint256 amount);

    // 7. Modifiers
    modifier onlyOwner() {
        if (msg.sender != i_owner) revert MyContract__NotOwner();
        _;
    }

    // 8. Constructor
    constructor(address priceFeed) {
        i_owner = msg.sender;
        s_priceFeed = AggregatorV3Interface(priceFeed);
    }

    // 9. Receive Function
    receive() external payable {
        fund();
    }

    // 10. Fallback Function
    fallback() external payable {
        fund();
    }

    // 11. External Functions
    function fund() public payable {
        require(msg.value.getConversionRate(s_priceFeed) >= MINIMUM_USD, "Insufficient ETH");
        s_addressToAmountFunded[msg.sender] += msg.value;
        s_funders.push(msg.sender);
        emit Funded(msg.sender, msg.value);
    }

    function withdraw() external onlyOwner {
        for (uint256 i = 0; i < s_funders.length; i++) {
            s_addressToAmountFunded[s_funders[i]] = 0;
        }
        s_funders = new address ;
        (bool success, ) = i_owner.call{value: address(this).balance}("");
        require(success, "Withdraw failed");
        emit Withdrawn(i_owner, address(this).balance);
    }

    // 12. Public Functions
    function getMinimumUSD() public pure returns (uint256) {
        return MINIMUM_USD;
    }

    // 13. Internal Functions
    function _helperFunction() internal view returns (uint256) {
        return block.timestamp;
    }

    // 14. Private Functions
    function _logSomething() private {
        // internal bookkeeping
    }

    // 15. View / Pure (Getter) Functions
    function getOwner() public view returns (address) {
        return i_owner;
    }

    function getFunder(uint256 index) public view returns (address) {
        return s_funders[index];
    }

    function getAddressToAmountFunded(address funder) public view returns (uint256) {
        return s_addressToAmountFunded[funder];
    }

    function getPriceFeed() public view returns (AggregatorV3Interface) {
        return s_priceFeed;
    }
}
```

---

## 🧭 Simplified Function Order Convention

| Order | Function Type | Description                                         |
| ----- | ------------- | --------------------------------------------------- |
| 1     | `constructor` | Runs once at deployment                             |
| 2     | `receive()`   | Handles plain ETH transfers                         |
| 3     | `fallback()`  | Handles unknown functions or ETH with data          |
| 4     | `external`    | Callable only from outside                          |
| 5     | `public`      | Callable from inside & outside                      |
| 6     | `internal`    | Callable only from inside the contract              |
| 7     | `private`     | Callable only within the same contract              |
| 8     | `view`/`pure` | Read-only or logic-only functions, includes getters |

---

## 🔤 General Naming Conventions
| Element               | Convention                              | Example                         |
| --------------------- | --------------------------------------- | ------------------------------- |
| **Contracts**         | `PascalCase`                            | `FundMe`, `MyToken`             |
| **Functions**         | `camelCase`                             | `getPrice()`, `sendFunds()`     |
| **Variables**         | `camelCase`                             | `priceFeed`, `ownerAddress`     |
| **Constants**         | `UPPER_CASE_WITH_UNDERSCORES`           | `MINIMUM_USD`, `MAX_SUPPLY`     |
| **Immutable vars**    | Prefix with `i_`                        | `i_owner`                       |
| **Private variables** | Prefix with `s_` (state var)            | `s_priceFeed`, `s_funders`      |
| **Custom errors**     | `PascalCase`, prefix with contract name | `FundMe__NotOwner()`            |
| **Events**            | `PascalCase`                            | `FundsReceived`, `WinnerPicked` |

---

Here's your extended and formatted section for `resources-notes.md` or Discord, including beginner-friendly `vm.*` commands:

---

## 🧪 Testing Style (Based on Cyfrin’s FundMe)

> 📁 Example: `test/FundMeTest.t.sol`

### ✅ Highlights:

* Uses `MockV3Aggregator` to simulate the **Chainlink price feed**
* Uses `vm.prank(...)` to simulate **calls from non-msg.sender accounts**
* Validates key behaviors: **funding**, **owner-only withdrawals**, **reverts**

---

### 🔧 Common `vm.` Cheat Sheet for Foundry Tests

| Command                     | Description                                                              |
| --------------------------- | ------------------------------------------------------------------------ |
| `vm.prank(address)`         | Next call is made from `address` (changes `msg.sender`)                  |
| `vm.startPrank(address)`    | Start a prank for multiple calls from `address`                          |
| `vm.stopPrank()`            | Stop a running `startPrank()`                                            |
| `vm.expectRevert()`         | Expect the next call to revert (optionally with a specific message)      |
| `vm.deal(address, amount)`  | Set ETH balance of `address`                                             |
| `vm.warp(timestamp)`        | Fast-forward block timestamp (e.g., `vm.warp(block.timestamp + 1 days)`) |
| `vm.roll(blockNumber)`      | Change the current block number                                          |
| `vm.mockCall(...)`          | Mock a low-level external contract call                                  |
| `vm.recordLogs()`           | Start recording logs for event testing                                   |
| `vm.load(addr, slot)`       | Load raw storage from a contract                                         |
| `vm.store(addr, slot, val)` | Write raw storage into a contract (use carefully!)                       |

---

### 📌 Example — `vm.mockCall(...)`

```solidity
vm.mockCall(
    address(mockFeed),
    abi.encodeWithSelector(AggregatorV3Interface.latestRoundData.selector),
    abi.encode(0, 2000e8, 0, 0, 0) // mock price data
);
```

* Mocks a call to `latestRoundData()`
* Useful when you don’t want to deploy or configure a real oracle mock
* Can simulate any external contract call returning controlled values

---

### 🎲 Fuzzing 101 — Intro to Property-Based Testing

#### 🧪 What is Fuzzing?

Fuzzing generates **random inputs** to test your functions under a wide range of scenarios. It helps uncover edge cases you might not think of manually.

#### 🚧 Tools:

| Command                  | Description                                          |
| ------------------------ | ---------------------------------------------------- |
| `vm.assume(condition)`   | Filter out invalid or unwanted inputs in a fuzz test |
| `bound(input, min, max)` | Force a fuzzed value to stay within a specific range |

---

#### ✍️ Example Fuzz Test with `vm.assume` + `bound`

```solidity
function test_FuzzFund(uint256 amount) public {
    // Only allow fuzzed inputs in the valid range
    vm.assume(amount > 1e18 && amount < 100 ether);

    vm.prank(user);
    fundMe.fund{value: amount}();

    uint256 funded = fundMe.getAddressToAmountFunded(user);
    assertEq(funded, amount);
}
```

Or using `bound()` instead:

```solidity
function test_FuzzFund(uint256 amount) public {
    amount = bound(amount, 1 ether, 100 ether);

    vm.prank(user);
    fundMe.fund{value: amount}();

    assertEq(fundMe.getAddressToAmountFunded(user), amount);
}
```

---

#### 📌 **Pro Tips:**

* Use `assume` to avoid invalid test cases.
* `bound` is simpler for numeric ranges — especially when testing balance limits or loop conditions.
* Fuzzing is most powerful when testing **invariants**, like "the contract’s balance should never go below 0."

---

### 🧬 Invariant Testing (Property Testing)

**Invariant tests** check that **a condition always holds**—no matter what sequence of actions the fuzzer tries.

#### 🔍 Use Case:

Instead of testing one function with random inputs, Foundry will **call many functions in random order** and check if a rule (invariant) is ever violated.

#### 🧪 Example:

```solidity
contract MyInvariantTest is Test {
    MyContract my;

    function setUp() public {
        my = new MyContract();
    }

    function invariant_totalIsAlwaysNonNegative() public {
        assert(my.total() >= 0);
    }
}
```

#### 🔁 Fuzzer will:

* Randomly call public/external functions on `my`
* After each sequence, it checks that `total() >= 0`

📌 Great for catching **unexpected side effects** and **state corruption**.

---

### 🧩 `cheatcodes.json` (Cheatcodes Documentation)

Foundry provides powerful tools for testing via **VM cheatcodes**, like `vm.prank`, `vm.mockCall`, etc.

📁 The file [`cheatcodes.json`](https://book.getfoundry.sh/cheatcodes/) is an **official list of all cheatcodes**, including:

| Cheatcode         | Purpose                             |
| ----------------- | ----------------------------------- |
| `vm.expectRevert` | Expect a revert during a test       |
| `vm.deal`         | Set ETH balance of an address       |
| `vm.warp`         | Set the next block’s timestamp      |
| `vm.mockCall`     | Mock external calls and return data |
| `vm.assume`       | Filter invalid fuzzed inputs        |

🛠 You can access the full doc here:
👉 [https://book.getfoundry.sh/cheatcodes/](https://book.getfoundry.sh/cheatcodes/)

---

## :brain: **What is a REPL?**

**REPL** = **Read–Eval–Print Loop**

It’s an interactive environment where you:

1. **Type code** (e.g., `2 + 2`)
2. It **executes** that code
3. It **prints** the result (e.g., `4`)
4. It **waits** for your next command

---

## :tools: **What is Chisel (for Solidity)?**

**Chisel** is a REPL for **Solidity** — like a **Solidity playground in your terminal**.

* Great for **trying out small snippets**, like variables, structs, math, mappings, etc.
* No need to **create full smart contracts**, write `.sol` files, or compile anything.
* Useful for **debugging**, **learning**, or **quick testing**.

---

### :bulb: Analogy

* Python → `python` in terminal
* JavaScript → `node` REPL
* Solidity → `chisel`

---

### :test_tube: Example usage

```bash
chisel
```

Then in the interactive prompt:

```solidity
> uint a = 10;
> a * 2
20
```

## 🧪 Anvil Seed vs Forge Test Seed
### :pushpin: What’s Happening:

By default, **Anvil uses the same seed** unless you explicitly specify one. So:

* Every time you manually start Anvil from the terminal, **contracts get the same addresses** (as long as the deployment order stays the same).
* This is convenient for manual testing and development with **hardcoded addresses**.

However, `forge test`:

* Starts its **own internal Anvil-like fork**, and by default uses a **random seed** each time.
* So deployed contract addresses **change every time**, breaking hardcoded assumptions.

---

### :jigsaw: Solutions

:white_check_mark: **Set a custom seed in Anvil:**

```bash
anvil --seed 123
```

> Any number works. This makes addresses deterministic again.

:white_check_mark: **Set seed for `forge test`:**

```bash
forge test --fork-url http://127.0.0.1:8545 --ffi --fork-block-number 18000000 --fork-seed 123
```

In practice, people more often use something like:

```bash
forge test --sender 0x... --gas-limit 10000000
```

Or if you’re using **Forge’s built-in Anvil**, pass the seed like this:

```bash
forge test --anvil --fork-url http://... --anvil-args "--seed 123"
```

---

### :brain: Recommendations

* **Avoid hardcoding addresses** that are determined during deployment.
  Instead, save them to variables directly in your scripts or tests.

* **Pass the seed explicitly** when you want deterministic results.

* Use `console.log(...)` or [`console2.sol`](https://github.com/foundry-rs/forge-std/blob/master/src/console2.sol) to **print addresses during deployment**, so you can copy them easily.

---

## 🔗 Contract Address Passing Options

| Method                       | On-Chain? | Notes                                           |
| ---------------------------- | --------- | ----------------------------------------------- |
| Constructor argument passing | ✅ Yes     | Use when deploying contracts in a single script |
| `.env` or shell              | ❌ No      | Flexible across deploy stages                   |
| JSON save/load               | ❌ No      | Good for multi-stage testing/deployment         |

---

## 🧱 Abstract Contracts

Abstract contracts can:

* Contain unimplemented functions (like interfaces)
* Also contain logic and state
* Be used as inheritance templates

```solidity
abstract contract MyAbstract {
    function doSomething() public virtual;
}
```

---

## ⚙️ `virtual` & `override`

Used for inheritance control:

```solidity
contract A {
    function foo() public virtual {}
}

contract B is A {
    function foo() public override {}
}
```

For multiple inheritance:

```solidity
contract C is A, B {
    function foo() public override(A, B) {}
}
```

---

## ⚙️ Assembly & Yul Basics

### 🧬 Yul: A Low-Level Language

| Opcode         | Description                      |
| -------------- | -------------------------------- |
| `sstore(k, v)` | Store `v` in slot `k`            |
| `sload(k)`     | Load value from slot `k`         |
| `mstore(o, v)` | Store `v` in memory offset `o`   |
| `return(o, s)` | Return `s` bytes from offset `o` |

### ✍️ Example

```solidity
function store(uint x) external {
    assembly {
        sstore(0x00, x)
    }
}

function load() external view returns (uint x) {
    assembly {
        x := sload(0x00)
    }
}
```

> ⚠️ Use `assembly` and `Yul` only when necessary — prone to bugs and harder to audit.

---

If you're building your portfolio, you can add sections for:

* ✅ Security patterns (e.g. reentrancy guard, access control)
* 📚 Reading notes from audits
* 🔍 Gas optimization tricks
* 🧪 Fuzzing, invariant testing, etc.

Would you like a sample README or repo structure suggestion too?
