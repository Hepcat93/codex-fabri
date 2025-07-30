# 📄 Bash Commands

A quick-reference guide to Bash/Zsh shell navigation and custom `.sh` scripting — focused on improving command-line efficiency and automating common Foundry development tasks.
Originally shared on the (🔥⚔🛡 ‹𝔽𝕠𝕦𝕟𝕕𝕣𝕪 & 𝔽𝕦𝕣𝕪› 🛡⚔🔥) Discord server for smart-contract audit study.

---

## 🧭 Terminal Navigation Shortcuts

> Use these shortcuts to move around quickly in your terminal (bash/zsh):

| Action                      | Shortcut   |
| --------------------------- | ---------- |
| Go to beginning of the line | `Ctrl + A` |
| Go to end of the line       | `Ctrl + E` |
| Move backward one word      | `Alt + B`  |
| Move forward one word       | `Alt + F`  |
| Move backward one character | `Ctrl + B` |
| Move forward one character  | `Ctrl + F` |

---

## 🛠️ Writing Bash Scripts for Foundry

### 🧾 Example: `foundry-menu.sh`

```bash
#!/bin/bash

# Exit the script on any error
set -e

echo "==========================="
echo "Foundry Script Menu"
echo "==========================="
echo "1) Start anvil"
echo "2) Compile contract"
echo "3) Run tests"
echo "4) Deploy contract"
echo "5) Send 1 ETH to contract"
echo "6) Check contract balance"
echo "0) Exit"
echo "==========================="

read -p "Choose an option: " choice

case $choice in
  1)
    echo "▶️ Starting anvil..."
    anvil
    ;;
  2)
    echo "🛠 Compiling..."
    forge build
    ;;
  3)
    echo "🧪 Running tests..."
    forge test
    ;;
  4)
    echo "🚀 Deploying..."
    forge script script/DeployFundMe.s.sol:DeployFundMe \
      --rpc-url http://127.0.0.1:8545 \
      --private-key $PRIVATE_KEY \
      --broadcast
    ;;
  5)
    echo "💸 Sending 1 ETH..."
    cast send 0xYourContractAddress "fund()" \
      --value 1ether \
      --rpc-url http://127.0.0.1:8545 \
      --private-key $PRIVATE_KEY
    ;;
  6)
    echo "💰 Contract balance:"
    cast balance 0xYourContractAddress \
      --rpc-url http://127.0.0.1:8545 | cast from-wei
    ;;
  0)
    echo "👋 Goodbye!"
    exit 0
    ;;
  *)
    echo "❌ Invalid option"
    ;;
esac
```

---

### ✅ How to Use the Script

1. Save it as `foundry-menu.sh`

2. Make it executable:

   ```bash
   chmod +x foundry-menu.sh
   ```

3. Run it:

   ```bash
   ./foundry-menu.sh
   ```

---

### 🔁 Optional: Run from Anywhere

Move it to a directory in your `$PATH`, e.g.:

```bash
mv foundry-menu.sh /usr/local/bin/foundry-menu
```

Now you can run it globally:

```bash
foundry-menu
```

No need for `./` or full path.
