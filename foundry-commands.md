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

**3. Re-deploy to the Same Address:**

Use the same salt and init code. The address stays the same due to CREATE2.

---

## 🧪 Running a Single Test File

```bash
forge test --match-path test/OnlyThisTest.t.sol
```

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

"functionName(argumentType1,argumentType2,...)"

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
