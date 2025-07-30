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

### > 📁 Example: `test/FundMeTest.t.sol`

This is a **working and correct** test file for your `FundMe` contract, considering:

* Use of a **real mock contract** (`MockV3Aggregator`);
* Proper passing of the mock address to the `FundMe` constructor;
* **No use of `vm.broadcast`**, since you're calling `run()` inside tests — `broadcast` is only for real deployments, not test environments;
* Avoiding the attempt to call `DeployFundMe.run()` as a `view`, which fails if `run()` includes `startBroadcast`.

---

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Test, console} from "forge-std/Test.sol";
import {FundMe} from "../src/FundMe.sol";
import {MockV3Aggregator} from "../src/test/mocks/MockV3Aggregator.sol";

contract FundMeTest is Test {
    FundMe public fundMe;
    MockV3Aggregator public mockV3Aggregator;

    address public OWNER = address(1); // any valid address

    function setUp() public {
        vm.startPrank(OWNER); // pretend OWNER is deploying

        mockV3Aggregator = new MockV3Aggregator(8, 3000e8);
        fundMe = new FundMe(address(mockV3Aggregator));

        vm.stopPrank(); // stop impersonation
    }

    function testMinimumDollarIsFive() public {
        assertEq(fundMe.MINIMUM_USD(), 5e18);
    }

    function testOwnerIsDeployer() public {
        assertEq(fundMe.getOwner(), OWNER);
    }

    function testPriceFeedVersion() public {
        uint256 version = fundMe.getVersion();
        assertEq(version, mockV3Aggregator.version());
    }

    function testFundFailsBelowMinimum() public {
        vm.prank(OWNER);

        // 1 ETH = 3000 USD → 5 USD ≈ 0.001666 ETH
        // Sending 0.001 ETH should revert
        vm.expectRevert();
        fundMe.fund{value: 0.001 ether}();
    }

    function testFundSucceedsWithEnoughEth() public {
        vm.prank(OWNER);

        fundMe.fund{value: 0.002 ether}();
        uint256 amount = fundMe.getAddressToAmountFunded(OWNER);

        assertGt(amount, 0);
    }

    function testWithdrawByOwner() public {
        vm.prank(OWNER);
        fundMe.fund{value: 0.01 ether}();

        uint256 balanceBefore = OWNER.balance;

        vm.prank(OWNER);
        fundMe.withdraw();

        uint256 balanceAfter = OWNER.balance;

        assertGt(balanceAfter, balanceBefore); // funds returned
    }

    function testWithdrawFailsIfNotOwner() public {
        vm.prank(OWNER);
        fundMe.fund{value: 0.01 ether}();

        address notOwner = address(2);
        vm.prank(notOwner);

        vm.expectRevert(); // should fail due to onlyOwner
        fundMe.withdraw();
    }

    receive() external payable {}
}
```

---

#### 🔍 What this file tests:

* Initializes `FundMe` with a real `MockV3Aggregator`;
* Fakes that `OWNER` deployed the contract;
* Validates:

  * the `MINIMUM_USD` constant;
  * ownership behavior;
  * version from price feed;
  * rejection of underfunding;
  * successful funding above the threshold;
  * owner-only withdrawals;
  * rejection of unauthorized withdrawals.

---

#### 🔧 Bonus: if you want to use `DeployFundMe`

If you'd still like to use `DeployFundMe`, you'd need to **move the deployment logic out of `run()`** into a separate function like `createFundMe()`, and **avoid `startBroadcast`** inside that method.

This way, tests can call the deployment logic without `broadcast` conflicts.

#### ChatGPT's suggestion for the later self research:

Let me know if you'd like a clean refactor of `DeployFundMe` to make it testable this way — happy to show it.

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

## 🛠 **What is Chisel (for Solidity)?**

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

### ✍️ Example usage

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

## 🔗 Automatic Contract Address Passing Options

### :question: **Question**

> “Is importing the only way to automatically reference another contract’s address after deployment in a script?”

**Answer:**
Not at all — there are **multiple automated approaches** to pass or retrieve a deployed contract’s address, and **importing is not required**.

---

### :white_check_mark: **Three Main Approaches**

| # | Method                                     | On-Chain Compatible? | Off-Chain Involved? | Use Case                                                       |
| - | ------------------------------------------ | -------------------- | ------------------- | -------------------------------------------------------------- |
| 1 | **Direct assignment in the deploy script** | ✅ Yes                | ❌ No                | Single deploy script where contracts are deployed sequentially |
| 2 | **Environment variables (.env, shell)**    | ❌ No                 | ✅ Yes               | Reusable CLI deploys or CI pipelines                           |
| 3 | **JSON serialization/deserialization**     | ❌ No                 | ✅ Yes               | Multi-step deployments or test setups                          |

---

### :mag_right: **Detailed Breakdown**

---

#### :white_check_mark: 1. Direct assignment within the deploy script (on-chain)

You deploy one contract and immediately use its address in the next deployment:

```solidity
function run() external {
    vm.startBroadcast();

    MockV3Aggregator mock = new MockV3Aggregator(...);
    FundMe fund = new FundMe(address(mock));

    vm.stopBroadcast();
}
```

:green_circle: **Pros**:

* Fully on-chain and atomic
* Clean, self-contained

:red_circle: **Cons**:

* Not modular — only works if both contracts are deployed in the same script/run

---

#### :white_check_mark: 2. Passing address via shell or `.env` variables (off-chain)

You pass the address manually (or via a shell script):

```bash
export PRICE_FEED=0xMockAddress
forge script script/DeployFundMe.s.sol:DeployFundMe \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --constructor-args $PRICE_FEED
```

You can also load from `.env` using `vm.envAddress(...)` in Solidity:

```solidity
address priceFeed = vm.envAddress("PRICE_FEED");
FundMe fund = new FundMe(priceFeed);
```

:green_circle: **Pros**:

* Good for separating deployments across stages or chains
* Integrates well with CI/CD pipelines

:red_circle: **Cons**:

* Not on-chain — depends on external files or user input
* Risk of mismatches if `.env` is outdated or missing

---

#### :white_check_mark: 3. Serialize & deserialize address using JSON (off-chain)

You write the deployed address into a `.json` file, and read it later:

**Save (serialize) during deployment:**

```solidity
string memory json = vm.serializeAddress("deploy", "priceFeed", address(mock));
vm.writeJson(json, "./deployments/mock.json");
```

**Load (deserialize) in another script:**

```solidity
address priceFeed = vm.parseJsonAddress("./deployments/mock.json", ".deploy.priceFeed");
FundMe fund = new FundMe(priceFeed);
```

:green_circle: **Pros**:

* Great for multi-script, multi-chain deployments
* Enables reuse of previously deployed contract data

:red_circle: **Cons**:

* Depends on off-chain cheatcodes (`vm.*`)
* Only works inside Foundry scripts, not in production contracts

---

### :pushpin: **Serialization: What Is It?**

**Serialization** is the process of converting a data object (like a contract address) into a storable format (e.g. JSON).
**Deserialization** is the reverse — reading the data back into code.

Useful when:

* You need to store results of one deployment to use later
* Avoid hardcoding
* Maintain flexibility across different networks

---

### :white_check_mark: **Which Works On-Chain?**

| Method                     | Works Fully On-Chain? |
| -------------------------- | --------------------- |
| Environment variables      | ❌ No                  |
| JSON serialization         | ❌ No                  |
| Direct constructor passing | ✅ Yes                 |

Only **method #3** (direct address passing in Solidity) is **fully on-chain**, meaning it:

* Doesn’t depend on external files
* Can run inside a contract, not just in a script

---

### :brain: Best Practices

* Use **direct variable passing** when everything is in one deploy script or transaction.
* Use **`.env` or JSON serialization** when:

  * Deployments are split across steps or teams
  * You want modularity and reuse
  * You're integrating with CI/CD

:white_check_mark: All three methods can be **combined** in large projects.

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
