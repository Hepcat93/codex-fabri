# 🛠️ Foundry CLI — Useful Commands & Concepts

This document collects practical examples of Foundry CLI tools like `forge`, `cast`, and `anvil`. Includes usage tips, test filters, forking techniques, and low-level EVM interactions like `CREATE2` and `selfdestruct`. Originally compiled and shared on the 🔥⚔🛡 ‹𝔽𝕠𝕦𝕟𝕕𝕣𝕪 & 𝔽𝕦𝕣𝕪› Discord server — my study group for smart contract auditing.

---

## 🔎 Checking Contract or EOA Balance

Command:

```bash
cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
  --rpc-url http://127.0.0.1:8545 | cast from-wei
````

* The `\` (backslash) lets you continue the command on the next line in a terminal.
* The `|` (pipe) pipes the output into `cast from-wei`, converting the balance from wei to ether.

---

## 🌐 Fork-running a Blockchain with Anvil

Launch a Sepolia fork using Alchemy:

```bash
anvil --fork-url https://eth-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_API_KEY
```

Replace `YOUR_ALCHEMY_API_KEY` with your actual Alchemy key.
You can use Infura or other node providers instead.

---

## 💣 Destroying and Re-deploying Contracts to the Same Address

### Deterministic Deployment with CREATE2

The formula for address:

```text
address = keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))[12:]
```

Use cases:

* Precomputing addresses
* Meta-transactions
* Upgradable proxies

### Selfdestruct

* Deletes contract code and storage
* Sends balance to a specified address
* Frees the address for reuse (but **not** tx history)

---

### ✅ Full Workflow Example (Foundry)

**1. Deploy with CREATE2:**

```bash
forge script script/DeployWithCreate2.s.sol \
  --rpc-url http://localhost:8545 \
  --broadcast \
  --sig "deploy(bytes32)" \
  0x1234567890000000000000000000000000000000000000000000000000000000
```

Inside the script:

```solidity
new MyContract{salt: salt}();
```

---

**2. Destroy the Contract:**

```bash
cast send 0xYourDeployedAddress "destroy(address)" 0xRecipient \
  --rpc-url http://localhost:8545 \
  --private-key $PRIVATE_KEY
```

---

**3. Re-deploy to the Same Address (DISABLED WITHIN THE NEWEST EIP's):**

Use the same salt and init code. The address stays the same due to CREATE2.

---

## 🧪 A **compact and convenient table** for working with `forge test` during development and audits:

| Command / Flag                                 | What it does                                  | When to use it                                   |
| ---------------------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| `forge test`                                   | Runs **all tests** in the `test/` folder      | Quick run of the full test suite                 |
| `forge test --match-contract ContractName`     | Runs tests for **a specific contract only**   | When you want to focus on one contract           |
| `forge test --match-path test/FileName.t.sol`  | Runs tests **from a specific test file only** | When you have many files and need just one       |
| `forge test --match-test testFunctionName`     | Runs **a single test function**               | For quick checking of a single scenario          |
| `forge test --match-contract X --match-test Y` | Combines both contract and test filtering     | To run a specific test in a specific contract    |
| `forge coverage`                               | Displays **code coverage (%)**                | To verify how complete your test coverage is     |
| `forge coverage --match-contract ContractName` | Shows coverage **for one specific contract**  | Useful for focused analysis                      |
| `-vvvv`                                        | Enables **very detailed logs**                | For debugging complex cases, reverts, cheatcodes |
| `--gas-report`                                 | Displays **gas usage per test**               | For gas optimization and identifying heavy ops   |
| `--fork-url <RPC>`                             | Runs tests on a **mainnet/testnet fork**      | To test real-world integrations, e.g. DeFi       |
| Fuzzing (parameterized tests)                  | Foundry automatically feeds random values     | For checking invariants and robustness of logic  |

---

💡 **Audit Tip:**

* To test **one contract only** → use `--match-contract` + `--match-test`.
* To analyze **code coverage** → use `forge coverage`.
* For **stress and invariant testing** → combine fuzzing with cheatcodes.

---

## 🧰 Mocking Calls Inside a Contract & Understanding Function Selectors

### Example: `vm.mockCall`

```solidity
bytes4 selector = AggregatorV3Interface.version.selector;

vm.mockCall(
  address(1),
  abi.encodeWithSelector(selector),
  abi.encode(uint256(1))
);
```

---

### What Is a Selector?

A function selector is the first **4 bytes** of the Keccak-256 hash of the function signature.

#### Signature:

The selector is derived from the function signature, which is:

```solidity
functionName(argumentType1,argumentType2,...)
```

For example:

function version() external view returns (uint256);

```solidity
"version()"
```

#### Hash:

To get the selector, Solidity takes the Keccak-256 hash of the function signature and truncates it to the first 4 bytes.

```solidity
bytes4 selector = bytes4(keccak256("version()"));
```

#### Example Output:

```text
0x54fd4d50
```

---

### Foundry Shortcut:

```bash
cast sig "version()"
# → 0x54fd4d50
```

---

### TL;DR:

* Selector = `keccak256("signature")[:4]`
* It tells the EVM which function to call
* `vm.mockCall` uses it to simulate return values

---

## 🧩 `forge fmt`

**Purpose:** formats Solidity code according to standard style rules.
**Analogy:** similar to `prettier` or `rustfmt` in other languages.

**Example:**

```bash
forge fmt
```

**What it does:**

* Automatically aligns indentation, spaces, and line breaks.
* Normalizes syntax for a consistent code style (handy before committing).
* By default, formats all `.sol` files in the project.
* You can also format specific files:

  ```bash
  forge fmt src/Contract.sol
  ```

**Configuration:**
Formatting is configured in the `foundry.toml` file, under the `[fmt]` section.
Example:

```toml
[fmt]
line_length = 100
tab_width = 4
bracket_spacing = true
```

---

## 🧮 `forge snapshot`

**Purpose:** creates a **snapshot** of gas usage for all tests.

**Example:**

```bash
forge snapshot
```

**What it does:**

* Runs all tests (`forge test`) and records the average gas cost of each test.
* Saves the results to a `.gas-snapshot` file in the project root.

Example contents:

```
testDeposit() (gas: 51234)
testWithdraw() (gas: 48210)
```

**Why it’s useful:**

* To track gas consumption changes between commits.
* Very convenient in CI/CD pipelines — helps detect performance regressions.

**Compare with the previous snapshot:**

```bash
forge snapshot --diff
```

shows which tests have become more expensive in terms of gas.

---

## Here’s a **compact and convenient table** for working with `forge test` during development and audits:

| Command / Flag                                 | What it does                                  | When to use it                                   |
| ---------------------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| `forge test`                                   | Runs **all tests** in the `test/` folder      | Quick run of the full test suite                 |
| `forge test --match-contract ContractName`     | Runs tests for **a specific contract only**   | When you want to focus on one contract           |
| `forge test --match-path test/FileName.t.sol`  | Runs tests **from a specific test file only** | When you have many files and need just one       |
| `forge test --match-test testFunctionName`     | Runs **a single test function**               | For quick checking of a single scenario          |
| `forge test --match-contract X --match-test Y` | Combines both contract and test filtering     | To run a specific test in a specific contract    |
| `forge coverage`                               | Displays **code coverage (%)**                | To verify how complete your test coverage is     |
| `forge coverage --match-contract ContractName` | Shows coverage **for one specific contract**  | Useful for focused analysis                      |
| `-vvvv`                                        | Enables **very detailed logs**                | For debugging complex cases, reverts, cheatcodes |
| `--gas-report`                                 | Displays **gas usage per test**               | For gas optimization and identifying heavy ops   |
| `--fork-url <RPC>`                             | Runs tests on a **mainnet/testnet fork**      | To test real-world integrations, e.g. DeFi       |
| Fuzzing (parameterized tests)                  | Foundry automatically feeds random values     | For checking invariants and robustness of logic  |

---

💡 **Audit Tip:**

* To test **one contract only** → use `--match-contract` + `--match-test`.
* To analyze **code coverage** → use `forge coverage`.
* For **stress and invariant testing** → combine fuzzing with cheatcodes.

---
