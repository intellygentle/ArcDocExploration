# 🟢 Part 1: Hello Architect
**Deploy your first smart contract on Arc Chain using Termux + Ubuntu.**

By the end of this guide, you will have created, tested, and interacted with a live smart contract on the Arc Testnet—all from your mobile phone.


### 1️⃣ Setting up the Environment
First, enter your Ubuntu container and install the necessary tools.

```
pkg update && pkg upgrade -y
pkg install proot-distro curl -y
proot-distro install ubuntu
proot-distro login ubuntu
apt update && apt upgrade -y
```

### 2️⃣ Install Foundry
Foundry is the toolkit we use to compile and deploy. Run this entire block:

```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
```

**Verify installation:**
 `forge --version`
 `cast --version`

### 3️⃣ Initialize the Project
We will create a fresh directory and clean out the default "Counter" files.

```
 `forge init arc-tutorial && cd arc-tutorial`
```

### Remove default file
```
rm src/Counter.sol test/Counter.t.sol
rm -rf script
```

### 4️⃣ Create the Contract
Copy this entire block to write the `HelloArchitect.sol` file:

```solidity
cat > src/HelloArchitect.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

contract HelloArchitect {
    string private greeting;
    event GreetingChanged(string newGreeting);

    constructor() {
        greeting = "Hello Architect!";
    }

    function setGreeting(string memory newGreeting) public {
        greeting = newGreeting;
        emit GreetingChanged(newGreeting);
    }

    function getGreeting() public view returns (string memory) {
        return greeting;
    }
}
EOF
```

### 5️⃣ Create the Test File
Run this to create your unit tests:

```solidity
cat > test/HelloArchitect.t.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "forge-std/Test.sol";
import "../src/HelloArchitect.sol";

contract HelloArchitectTest is Test {
    HelloArchitect helloArchitect;

    function setUp() public {
        helloArchitect = new HelloArchitect();
    }

    function testInitialGreeting() public view {
        string memory expected = "Hello Architect!";
        assertEq(helloArchitect.getGreeting(), expected);
    }

    function testSetGreeting() public {
        string memory newGreeting = "Welcome to Arc Chain!";
        helloArchitect.setGreeting(newGreeting);
        assertEq(helloArchitect.getGreeting(), newGreeting);
    }
}
EOF
```

**Run the test:**
```
forge test
```

*You should see: "Test result: ok. 2 passed"* ✅

### 6️⃣ Network Configuration
Now, we set up your wallet and the Arc RPC. 
**Note:** Replace `0x_your_private_key` with your actual private key.

```bash
cat >> ~/.bashrc << 'EOF'
export ARC_TESTNET_RPC_URL="https://rpc.testnet.arc.network"
export PRIVATE_KEY="0x_your_evm_wallet_privatekey_here"
EOF
source ~/.bashrc
```
*Make sure your wallet has Testnet USDC from the Circle Faucet!*

### 7️⃣ Deploy to Arc Testnet
Execute the deployment:

```bash
forge create src/HelloArchitect.sol:HelloArchitect \
  --rpc-url $ARC_TESTNET_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast
```

**Copy the "Deployed to" address.** We need to save it:

```bash
# Replace 0x_address with your actual deployed address
export HELLO_ADDR="0x_your_deployed_address"
echo "export HELLO_ADDR=\"$HELLO_ADDR\"" >> ~/.bashrc
source ~/.bashrc
```

### 8️⃣ Interact with the Blockchain
**Read the greeting (Gasless):**
```bash
cast call $HELLO_ADDR "getGreeting()(string)" --rpc-url $ARC_TESTNET_RPC_URL
```

**Update the greeting (Requires Gas):**
```bash
cast send $HELLO_ADDR "setGreeting(string)" "Glad to explore Arc using termux right from mobile phone" --rpc-url $ARC_TESTNET_RPC_URL --private-key $PRIVATE_KEY
```

**Verify the update:**
```bash
cast call $HELLO_ADDR "getGreeting()(string)" --rpc-url $ARC_TESTNET_RPC_URL
```

---

## 🎉 Success!
You just deployed a smart contract on **Arc Chain** from an Android phone. You are officially an Onchain Architect.

**[Next: Master Batching Payouts (Part 2) →](../README.md)**
