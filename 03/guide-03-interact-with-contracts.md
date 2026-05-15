# Guide 03 — Interact with Smart Contracts on Arc Testnet (Termux)

**A direct continuation of [Guide 02](https://github.com/intellygentle/ArcDocExploration/tree/main/02).** This guide teaches you how to **deploy** four contract types, **interact** with them (mint, transfer, airdrop), and **verify** everything on the explorer — all from your Android phone.

---

## What You Will Learn

By the end of this guide you will understand and do:

- How **Circle** and **Arc** work together (the tech behind everything)
- A clean `.env` filing system so you never confuse contract IDs again
- Deploy ERC-20, ERC-721, ERC-1155, and Airdrop contracts
- Mint tokens and NFTs
- Transfer tokens safely (including the ERC-721 token ID fix)
- Approve and execute airdrops without silent failures
- Check balances and transaction status on the Arc explorer

---

## How Circle + Arc Work Together (The Tech)

Before we write code, let's understand **what is happening under the hood**. This will save you hours of confusion later.

### The Two Players

| Player | What It Does | Analogy |
|--------|-------------|---------|
| **Circle** | Manages your wallets, signs transactions with your private keys, talks to the blockchain for you | A bank that holds your keys and sends money on your behalf |
| **Arc** | The blockchain (testnet) where your contracts live and transactions get recorded | The postal system that delivers and records every letter |

### The Flow of Every Transaction

```
Your Code (TypeScript)
    ↓
Circle SDK (sends API request to Circle's servers)
    ↓
Circle Servers (signs your transaction with your wallet's private key)
    ↓
Arc Testnet (processes the transaction, updates the blockchain)
    ↓
Arc Explorer (shows you what happened at testnet.arcscan.app)
```

You **never touch private keys directly**. Circle holds them securely. Your code just says "mint 1 token" and Circle handles the cryptographic signing.

### Two Types of IDs

Every contract has **two identities**:

| ID Type | Example | Where It Lives | What It's For |
|---------|---------|---------------|---------------|
| **Circle Contract ID** | `019e0373-35de-7882-...` | Circle's database | Circle's internal reference to track your contract |
| **Blockchain Address** | `0x0d05a94dbf235f...` | Arc blockchain | The actual address on the blockchain where your contract lives |

You need **both**. Circle IDs let Circle look up your contract. Blockchain addresses let the blockchain know where to send transactions.

### Transaction States

Every transaction goes through these states:

```
INITIATED  →  PENDING  →  COMPLETE
   (Circle      (On Arc,        (Done! Check
    received     waiting for     the explorer
    your         confirmation)   for details)
    request)
```

You **must wait for COMPLETE** before doing the next step. If you mint and then immediately transfer, the transfer will fail because the mint hasn't happened yet on the blockchain.

---

## Prerequisites

You **must** have completed [Guide 02](https://github.com/intellygentle/ArcDocExploration/tree/main/02). You need:

- Android phone with Termux + Ubuntu set up
- Circle developer account and API key
- Your `hello-arc` project folder with packages installed
- Test USDC in your wallet (from [faucet.circle.com](https://faucet.circle.com))

**Important:** This guide requires **Node.js 20 or newer**. The `--env-file` flag we use to load `.env` variables was added in Node.js 20. Check your version:

```bash
node --version
```

If it shows less than `v20.0.0`, update Node.js inside Ubuntu:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
```

---

## Step 1: Return to Your Environment

Open **Termux** and run:

```bash
proot-distro login ubuntu
cd ~/hello-arc
```

You are back where Guide 02 left you.

---

## Step 2: Create Your `.env` File

The `.env` file is where you store **all** your credentials and contract addresses in one place. Think of it as a form — you print blank forms now, then fill them in as you deploy each contract.

### 2.1 Create the File

```bash
cat << 'EOF' > .env
# ==========================================
# YOUR CIRCLE CREDENTIALS (from Guide 02)
# ==========================================
CIRCLE_API_KEY=PASTE_YOUR_API_KEY_HERE
CIRCLE_ENTITY_SECRET=PASTE_YOUR_ENTITY_SECRET_HERE
WALLET_ID=PASTE_YOUR_WALLET_ID_HERE
WALLET_ADDRESS=0xPASTE_YOUR_WALLET_ADDRESS_HERE

# ==========================================
# RECIPIENT (who receives transferred tokens)
# Use a second wallet address, or your own
# ==========================================
RECIPIENT_WALLET_ADDRESS=0xPASTE_RECIPIENT_ADDRESS_HERE

# ==========================================
# ERC-20 TOKEN CONTRACT
# Fill after deploying your ERC-20
# ==========================================
ERC20_CONTRACT_ID=
ERC20_CONTRACT_ADDRESS=
ERC20_TRANSACTION_ID=

# ==========================================
# ERC-721 NFT CONTRACT
# Fill after deploying your ERC-721
# ==========================================
ERC721_CONTRACT_ID=
ERC721_CONTRACT_ADDRESS=
ERC721_TRANSACTION_ID=

# ==========================================
# ERC-1155 MULTI-TOKEN CONTRACT
# Fill after deploying your ERC-1155
# ==========================================
ERC1155_CONTRACT_ID=
ERC1155_CONTRACT_ADDRESS=
ERC1155_TRANSACTION_ID=

# ==========================================
# AIRDROP CONTRACT
# Fill after deploying your Airdrop
# ==========================================
AIRDROP_CONTRACT_ID=
AIRDROP_CONTRACT_ADDRESS=
AIRDROP_TRANSACTION_ID=

# ==========================================
# GENERIC TRANSACTION CHECKER
# Copy any transaction ID here to check its status
# ==========================================
TRANSACTION_ID=
EOF
```

### 2.2 Fill In Your Credentials

Edit the file to add your real Circle credentials:

```bash
nano .env
```

Replace these placeholders with your real values from Guide 02:
- `PASTE_YOUR_API_KEY_HERE` → your Circle API key
- `PASTE_YOUR_ENTITY_SECRET_HERE` → your entity secret
- `PASTE_YOUR_WALLET_ID_HERE` → your wallet UUID
- `PASTE_YOUR_WALLET_ADDRESS_HERE` → your wallet address (starts with `0x`)
- `PASTE_RECIPIENT_ADDRESS_HERE` → a second wallet address (or your own)

**Save and exit:** `Ctrl+O` → Enter → `Ctrl+X`

> **Why so many empty lines?**
> Think of `.env` as a form. Right now you are printing blank forms. As you deploy each contract, you will "fill in the form" with real IDs and addresses. This prevents you from accidentally using the wrong contract.

> **Security warning:** Never share your `.env` file or commit it to GitHub. It contains your private API key and entity secret.

---

## Step 3: Add npm Scripts

These scripts let you run your TypeScript files with one simple command. Run these **one by one**:

```bash
npm pkg set scripts.deploy-erc20="tsx --env-file=.env deploy-erc20.ts"
npm pkg set scripts.deploy-erc721="tsx --env-file=.env deploy-erc721.ts"
npm pkg set scripts.deploy-erc1155="tsx --env-file=.env deploy-erc1155.ts"
npm pkg set scripts.deploy-airdrop="tsx --env-file=.env deploy-airdrop.ts"
npm pkg set scripts.get-contract="tsx --env-file=.env get-contract.ts"
npm pkg set scripts.check-tx="tsx --env-file=.env check-transaction.ts"
npm pkg set scripts.check-erc20="tsx --env-file=.env check-erc20-tx.ts"
npm pkg set scripts.check-erc721="tsx --env-file=.env check-erc721-tx.ts"
npm pkg set scripts.check-erc1155="tsx --env-file=.env check-erc1155-tx.ts"
npm pkg set scripts.check-airdrop="tsx --env-file=.env check-airdrop-tx.ts"
npm pkg set scripts.interact-erc20="tsx --env-file=.env interact-erc20.ts"
npm pkg set scripts.interact-erc721="tsx --env-file=.env interact-erc721.ts"
npm pkg set scripts.interact-erc1155="tsx --env-file=.env interact-erc1155.ts"
npm pkg set scripts.approve-airdrop="tsx --env-file=.env approve-airdrop.ts"
npm pkg set scripts.interact-airdrop="tsx --env-file=.env interact-airdrop.ts"
```

**What is `--env-file=.env`?** It tells Node.js to load all variables from your `.env` file into `process.env`. That's how your scripts read `CIRCLE_API_KEY`, `WALLET_ID`, etc. without hardcoding them.

---

## Step 4: Create Helper Scripts

### 4.1 The Get-Contract Script

This script takes a Circle Contract ID and fetches the blockchain address for you.

```bash
cat << 'EOF' > get-contract.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const client = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const type = process.env.CONTRACT_TYPE;

  if (!type) {
    console.error("❌ Please set CONTRACT_TYPE before running this script.");
    console.error("   Example: CONTRACT_TYPE=ERC20 npm run get-contract");
    console.error("   Options: ERC20, ERC721, ERC1155, AIRDROP");
    process.exit(1);
  }

  const contractIdKey = `${type}_CONTRACT_ID`;
  const contractId = process.env[contractIdKey];

  if (!contractId || contractId.trim() === "") {
    console.error(`❌ ${contractIdKey} is missing or empty in your .env file!`);
    console.error(`   Please deploy the ${type} contract first and paste its contractId into .env`);
    process.exit(1);
  }

  console.log(`🔍 Fetching ${type} contract details from Circle...\n`);

  const response = await client.getContract({ id: contractId });

  console.log("📋 Full Contract Response:");
  console.log(JSON.stringify(response.data, null, 2));

  const address = response.data?.contract?.contractAddress;
  if (address) {
    console.log(`\n✅ SUCCESS! Here is your ${type} blockchain address:`);
    console.log(`   ${type}_CONTRACT_ADDRESS=${address}`);
    console.log(`\n📝 Copy the line above and paste it into your .env file.`);
  } else {
    console.log("\n⚠️  Contract address not found. Deployment may still be pending.");
    console.log("   Wait 30 seconds and try again.");
  }
}

main().catch(console.error);
EOF
```

### 4.2 The Check-Transaction Scripts

These scripts check if a transaction is complete.

**check-transaction.ts** (generic — uses `TRANSACTION_ID` from `.env`):

```bash
cat << 'EOF' > check-transaction.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";
import { readFileSync } from "fs";

const envContent = readFileSync(".env", "utf8");
const env: Record<string, string> = {};
envContent.split("\n").forEach(line => {
  const [key, ...rest] = line.split("=");
  if (key && rest.length > 0) {
    env[key.trim()] = rest.join("=").trim();
  }
});

async function checkTransaction() {
  const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
    apiKey: env.CIRCLE_API_KEY,
    entitySecret: env.CIRCLE_ENTITY_SECRET,
  });

  console.log("Checking transaction status...");
  const transactionResponse = await circleDeveloperSdk.getTransaction({
    id: env.TRANSACTION_ID,
  });

  console.log(JSON.stringify(transactionResponse.data, null, 2));

  const state = transactionResponse.data?.transaction?.state;
  console.log(`Transaction state: ${state}`);

  if (state === "COMPLETE") {
    console.log("✅ Transaction completed successfully!");
  } else if (state === "PENDING") {
    console.log("⏳ Transaction still pending. Run again in 10-30 seconds.");
  }
}

checkTransaction().catch(error => {
  console.error("❌ Error:", error.message);
  process.exit(1);
});
EOF
```

**check-erc20-tx.ts** (uses `ERC20_TRANSACTION_ID`):

```bash
cat << 'EOF' > check-erc20-tx.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const txId = process.env.ERC20_TRANSACTION_ID;
  if (!txId) {
    console.error("❌ Set ERC20_TRANSACTION_ID in .env first");
    process.exit(1);
  }

  console.log("🔍 Checking ERC-20 Transaction...");
  console.log("Circle ID:", txId);

  try {
    const response = await client.getTransaction({ id: txId });
    console.log("\n📊 Status:", response.data.state);

    if (response.data.transactionHash) {
      const txHash = response.data.transactionHash;
      console.log("✅ On-chain txHash:", txHash);
      console.log(`\n🔗 View on Arc Testnet Explorer:`);
      console.log(`https://testnet.arcscan.app/tx/${txHash}`);
    } else {
      console.log("⏳ Transaction is still processing... Run this script again in 5-10 seconds.");
    }

    console.log("\nFull Response:");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) console.error("Details:", JSON.stringify(error.response.data, null, 2));
  }
}

main().catch(console.error);
EOF
```

**check-erc721-tx.ts** (uses `ERC721_TRANSACTION_ID`):

```bash
cat << 'EOF' > check-erc721-tx.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const txId = process.env.ERC721_TRANSACTION_ID;
  if (!txId) {
    console.error("❌ Set ERC721_TRANSACTION_ID in .env first");
    process.exit(1);
  }

  console.log("🔍 Checking ERC-721 Transaction...");
  console.log("Circle ID:", txId);

  try {
    const response = await client.getTransaction({ id: txId });
    console.log("\n📊 Status:", response.data.state);

    if (response.data.transactionHash) {
      const txHash = response.data.transactionHash;
      console.log("✅ On-chain txHash:", txHash);
      console.log(`\n🔗 View on Arc Testnet Explorer:`);
      console.log(`https://testnet.arcscan.app/tx/${txHash}`);
    } else {
      console.log("⏳ Transaction is still processing... Run this script again in 5-10 seconds.");
    }

    console.log("\nFull Response:");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) console.error("Details:", JSON.stringify(error.response.data, null, 2));
  }
}

main().catch(console.error);
EOF
```

**check-erc1155-tx.ts** (uses `ERC1155_TRANSACTION_ID`):

```bash
cat << 'EOF' > check-erc1155-tx.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const txId = process.env.ERC1155_TRANSACTION_ID;
  if (!txId) {
    console.error("❌ Set ERC1155_TRANSACTION_ID in .env first");
    process.exit(1);
  }

  console.log("🔍 Checking ERC-1155 Transaction...");
  console.log("Circle ID:", txId);

  try {
    const response = await client.getTransaction({ id: txId });
    console.log("\n📊 Status:", response.data.state);

    if (response.data.transactionHash) {
      const txHash = response.data.transactionHash;
      console.log("✅ On-chain txHash:", txHash);
      console.log(`\n🔗 View on Arc Testnet Explorer:`);
      console.log(`https://testnet.arcscan.app/tx/${txHash}`);
    } else {
      console.log("⏳ Transaction is still processing... Run this script again in 5-10 seconds.");
    }

    console.log("\nFull Response:");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) console.error("Details:", JSON.stringify(error.response.data, null, 2));
  }
}

main().catch(console.error);
EOF
```

**check-airdrop-tx.ts** (uses `AIRDROP_TRANSACTION_ID`):

```bash
cat << 'EOF' > check-airdrop-tx.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const txId = process.env.AIRDROP_TRANSACTION_ID;
  if (!txId) {
    console.error("❌ Set AIRDROP_TRANSACTION_ID in .env first");
    process.exit(1);
  }

  console.log("🔍 Checking Airdrop Transaction...");
  console.log("Circle ID:", txId);

  try {
    const response = await client.getTransaction({ id: txId });
    console.log("\n📊 Status:", response.data.state);

    if (response.data.transactionHash) {
      const txHash = response.data.transactionHash;
      console.log("✅ On-chain txHash:", txHash);
      console.log(`\n🔗 View on Arc Testnet Explorer:`);
      console.log(`https://testnet.arcscan.app/tx/${txHash}`);
    } else {
      console.log("⏳ Transaction is still processing... Run this script again in 5-10 seconds.");
    }

    console.log("\nFull Response:");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) console.error("Details:", JSON.stringify(error.response.data, null, 2));
  }
}

main().catch(console.error);
EOF
```

---

## Step 5: Deploy All Contracts (One by One)

For **each contract**, the workflow is identical:

1. **Run** the deploy script
2. **Copy** the `transactionId` and `contractId` from the output
3. **Paste** them into `.env` under the correct prefixed variables
4. **Wait** for the transaction to show `COMPLETE`
5. **Run** `get-contract` to retrieve the blockchain address
6. **Paste** the address into `.env`
7. Move to the next contract

---

### 5.1 Deploy ERC-20 Token

**What is ERC-20?** A standard for **fungible tokens** — tokens where every unit is identical and interchangeable. Think of it like dollars: every $1 bill is worth the same as every other $1 bill. USDC, DAI, and most cryptocurrencies are ERC-20 tokens.

```bash
cat << 'EOF' > deploy-erc20.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const circleContractSdk = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await circleContractSdk.deployContractTemplate({
    id: "a1b74add-23e0-4712-88d1-6b3009e85a86",
    blockchain: "ARC-TESTNET",
    name: "MyTokenContract",
    walletId: process.env.WALLET_ID!,
    templateParameters: {
      name: "MyToken",
      symbol: "MTK",
      defaultAdmin: process.env.WALLET_ADDRESS!,
      primarySaleRecipient: process.env.WALLET_ADDRESS!,
    },
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ ERC-20 Deployment started!");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF
```

Run it:

```bash
npm run deploy-erc20
```

**You will see output like:**

```json
{
  "transactionId": "019c...",
  "contractIds": ["019d..."]
}
```

**Do this immediately — edit `.env`:**

```bash
nano .env
```

Find the ERC-20 section and fill it in:

```
ERC20_CONTRACT_ID=019d...        ← from contractIds[0]
ERC20_TRANSACTION_ID=019c...     ← from transactionId
# Leave ERC20_CONTRACT_ADDRESS empty for now
```

**Save:** `Ctrl+O` → Enter → `Ctrl+X`

**Wait for the transaction to complete:**

```bash
npm run check-erc20
```

Keep running it every 10 seconds until it says `COMPLETE`.

**Get the blockchain address:**

```bash
CONTRACT_TYPE=ERC20 npm run get-contract
```

You will see a line like:
```
ERC20_CONTRACT_ADDRESS=0x2811...
```

Copy that line into your `.env` using `nano .env`.

**ERC-20 is fully registered!**

---

### 5.2 Deploy ERC-721 (NFT)

**What is ERC-721?** A standard for **non-fungible tokens (NFTs)** — tokens where each one is unique. Think of it like a house deed: every house has a unique address and value. Each NFT has a unique ID and can have its own metadata (image, name, description).

```bash
cat << 'EOF' > deploy-erc721.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const circleContractSdk = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await circleContractSdk.deployContractTemplate({
    id: "76b83278-50e2-4006-8b63-5b1a2a814533",
    blockchain: "ARC-TESTNET",
    name: "MyNFTContract",
    walletId: process.env.WALLET_ID!,
    templateParameters: {
      name: "MyNFT",
      symbol: "MNFT",
      defaultAdmin: process.env.WALLET_ADDRESS!,
      primarySaleRecipient: process.env.WALLET_ADDRESS!,
      royaltyRecipient: process.env.WALLET_ADDRESS!,
      royaltyPercent: 0.01,
    },
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ ERC-721 Deployment started!");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run deploy-erc721
```

**Update `.env` with the `ERC721_` prefixed values:**

```bash
nano .env
```

Fill in:
```
ERC721_CONTRACT_ID=...        ← from contractIds[0]
ERC721_TRANSACTION_ID=...     ← from transactionId
```

**Wait for COMPLETE:**

```bash
npm run check-erc721
```

**Get Contract Address:**

```bash
CONTRACT_TYPE=ERC721 npm run get-contract
```

Paste `ERC721_CONTRACT_ADDRESS=0x...` into `.env`.

---

### 5.3 Deploy ERC-1155 (Multi-Token)

**What is ERC-1155?** A standard for **multi-token contracts** — one contract that can hold many different token types, both fungible and non-fungible. Think of it like a vending machine: one machine holds many different products, each with its own ID and quantity. Game items, collectible sets, and batch-minted tokens often use ERC-1155.

```bash
cat << 'EOF' > deploy-erc1155.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const circleContractSdk = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await circleContractSdk.deployContractTemplate({
    id: "aea21da6-0aa2-4971-9a1a-5098842b1248",
    blockchain: "ARC-TESTNET",
    name: "MyMultiTokenContract",
    walletId: process.env.WALLET_ID!,
    templateParameters: {
      name: "MyMultiToken",
      symbol: "MMTK",
      defaultAdmin: process.env.WALLET_ADDRESS!,
      primarySaleRecipient: process.env.WALLET_ADDRESS!,
      royaltyRecipient: process.env.WALLET_ADDRESS!,
      royaltyPercent: 0.01,
    },
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ ERC-1155 Deployment started!");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run deploy-erc1155
```

Update `.env`:
```
ERC1155_CONTRACT_ID=...
ERC1155_TRANSACTION_ID=...
```

Wait for COMPLETE:
```bash
npm run check-erc1155
```

Get address:
```bash
CONTRACT_TYPE=ERC1155 npm run get-contract
```

Paste `ERC1155_CONTRACT_ADDRESS=0x...` into `.env`.

---

### 5.4 Deploy Airdrop Contract

**What is an Airdrop contract?** A special contract that **distributes tokens to many people at once**. Instead of sending tokens one by one (which costs gas for each transfer), an airdrop batches them into a single transaction. Projects use airdrops to send free tokens to community members, reward users, or distribute NFTs.

```bash
cat << 'EOF' > deploy-airdrop.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const circleContractSdk = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await circleContractSdk.deployContractTemplate({
    id: "13e322f2-18dc-4f57-8eed-4bddfc50f85e",
    blockchain: "ARC-TESTNET",
    name: "MyAirdropContract",
    walletId: process.env.WALLET_ID!,
    templateParameters: {
      defaultAdmin: process.env.WALLET_ADDRESS!,
    },
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Airdrop Deployment started!");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run deploy-airdrop
```

Update `.env`:
```
AIRDROP_CONTRACT_ID=...
AIRDROP_TRANSACTION_ID=...
```

Wait for COMPLETE:
```bash
npm run check-airdrop
```

Get address:
```bash
CONTRACT_TYPE=AIRDROP npm run get-contract
```

Paste `AIRDROP_CONTRACT_ADDRESS=0x...` into `.env`.

---

### Your `.env` Should Now Look Like This

By now, every field should be filled in. Your `.env` is a complete dashboard of your Arc Testnet empire.

```
CIRCLE_API_KEY=TEST_API_KEY:your_actual_key_here
CIRCLE_ENTITY_SECRET=your_actual_secret_here
WALLET_ID=your-wallet-uuid
WALLET_ADDRESS=0xYourWalletAddress
RECIPIENT_WALLET_ADDRESS=0xRecipientAddress

ERC20_CONTRACT_ID=019e0373-...
ERC20_CONTRACT_ADDRESS=0x0d05a94dbf...
ERC20_TRANSACTION_ID=f627fb92-...

ERC721_CONTRACT_ID=019e037a-...
ERC721_CONTRACT_ADDRESS=0x328c304345...
ERC721_TRANSACTION_ID=05c2271b-...

ERC1155_CONTRACT_ID=019e0380-...
ERC1155_CONTRACT_ADDRESS=0x5b937077d9...
ERC1155_TRANSACTION_ID=3c749e11-...

AIRDROP_CONTRACT_ID=019e0386-...
AIRDROP_CONTRACT_ADDRESS=0xbc77bb7349...
AIRDROP_TRANSACTION_ID=59ecc48e-...
```

---

## Step 6: Interact with Your Contracts

Now that every contract is registered in `.env`, the interaction scripts know exactly which contract to talk to.

### 6.1 ERC-20: Mint + Transfer

**What happens here:**
1. **Mint** — Creates 1 new token and gives it to your wallet. Think of it like a government printing new money.
2. **Transfer** — Sends 1 token from your wallet to the recipient. Think of it like handing cash to a friend.

**Important:** You must **wait for the mint to COMPLETE** before transferring. The transfer will fail if the tokens don't exist yet.

```bash
cat << 'EOF' > interact-erc20.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  if (!process.env.ERC20_CONTRACT_ADDRESS || process.env.ERC20_CONTRACT_ADDRESS.trim() === "") {
    console.error("❌ ERC20_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }
  if (!process.env.RECIPIENT_WALLET_ADDRESS || process.env.RECIPIENT_WALLET_ADDRESS.trim() === "") {
    console.error("❌ RECIPIENT_WALLET_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("🚀 ERC-20 Interaction: Mint + Transfer\n");
  console.log("💡 Tip: If transfer fails, make sure the mint transaction is COMPLETE first.\n");

  // STEP 1: MINT
  console.log("🖨️  Minting 1 token to your wallet...");
  const mintResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "1000000000000000000", // 1 token with 18 decimals
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Mint initiated!");
  console.log(`   Transaction ID: ${mintResponse.data?.id}`);
  console.log(`\n⏳ STOP HERE. Run 'npm run check-erc20' and wait for COMPLETE.`);
  console.log(`   Then come back and run this script again to do the transfer.\n`);

  // STEP 2: TRANSFER (only runs after mint is complete)
  console.log("📤 Transferring 1 token to recipient...");
  const transferResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "transfer(address,uint256)",
    abiParameters: [
      process.env.RECIPIENT_WALLET_ADDRESS!,
      "1000000000000000000", // 1 token with 18 decimals
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Transfer initiated!");
  console.log(`   Transaction ID: ${transferResponse.data?.id}`);
  console.log("\n🎉 Done! Check both transactions with npm run check-erc20");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc20
```

**What to expect:**
- You'll see the mint transaction ID
- **Wait for it to complete** before the transfer will work
- Check status: `npm run check-erc20`

> **What are "18 decimals"?**
> ERC-20 tokens use 18 decimal places by default (like how dollars use 2 decimal places for cents). So `1000000000000000000` = 1.000000000000000000 tokens. This allows tokens to be divided into very small fractions.

---

### 6.2 ERC-721: Mint an NFT

**What happens here:**
1. **Mint** — Creates a brand new NFT with a unique ID and assigns it to your wallet
2. The NFT has metadata stored on IPFS (a decentralized file system) — this is where the image, name, and description live

```bash
cat << 'EOF' > interact-erc721.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  if (!process.env.ERC721_CONTRACT_ADDRESS || process.env.ERC721_CONTRACT_ADDRESS.trim() === "") {
    console.error("❌ ERC721_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("🚀 ERC-721: Minting a new NFT\n");

  console.log("🎨 Minting NFT to your wallet...");
  const mintResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,string)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei",
    ],
    contractAddress: process.env.ERC721_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Mint initiated!");
  console.log(`   Transaction ID: ${mintResponse.data?.id}`);
  console.log("\n⏳ Wait for COMPLETE, then use the transfer script to send it to a friend.");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc721
```

**What to expect:**
- The mint creates NFT with Token ID `0` (the first one)
- Check status: `npm run check-erc721`

---

### 6.3 ERC-721: Transfer an NFT

**What happens here:**
- **Transfer** — Sends your NFT from your wallet to the recipient using `safeTransferFrom`
- You must specify the **Token ID** of the NFT you want to send

```bash
cat << 'EOF' > transfer-erc721.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const TOKEN_ID = "0";  // Token ID 0 = the first NFT you minted

  console.log(`📤 Transferring NFT #${TOKEN_ID}...`);
  console.log(`   From: ${process.env.WALLET_ADDRESS}`);
  console.log(`   To:   ${process.env.RECIPIENT_WALLET_ADDRESS}`);
  console.log(`   Contract: ${process.env.ERC721_CONTRACT_ADDRESS}\n`);

  const response = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "safeTransferFrom(address,address,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,           // from = owner (you)
      process.env.RECIPIENT_WALLET_ADDRESS!, // to = recipient
      TOKEN_ID,
    ],
    contractAddress: process.env.ERC721_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Transfer initiated!");
  console.log(`   Transaction ID: ${response.data?.id}`);
  console.log("\n⏳ Check status with npm run check-erc721");
  console.log("   On the explorer, look for 'Tokens Transferred' showing your NFT moving.");
}

main().catch(console.error);
EOF
```

Run:

```bash
npx tsx --env-file=.env transfer-erc721.ts
```

> **Why Token ID "0"?**
> When you minted the first NFT, the contract assigned it Token ID `0`. If you mint more NFTs, they get IDs `1`, `2`, `3`, etc. Change the `TOKEN_ID` variable to transfer a different NFT.

---

### 6.4 ERC-1155: Mint + Batch Transfer

**What happens here:**
1. **Mint** — Creates Token ID `0` in your ERC-1155 contract
2. **Batch Transfer** — Sends Token ID `0` to the recipient

**The big number explained:**
The first parameter in the mint is `115792089237316195423570985008687907853269984665640564039457584007913129639935`. This is `2^256 - 1` — the largest number the blockchain can hold (called "max uint256"). Passing it tells the ERC-1155 contract: "Create a brand new token ID." The contract assigns it ID `0`. For later mints, pass `0` and the contract creates IDs `1`, `2`, `3`, etc.

```bash
cat << 'EOF' > interact-erc1155.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  console.log("🚀 ERC-1155 Interaction: Mint + Batch Transfer\n");

  // MINT: Create Token ID 0 (pass max uint256 to create new ID)
  console.log("🖨️  Minting Token ID 0 (creating new token type)...");
  const mintResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,uint256,string,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "115792089237316195423570985008687907853269984665640564039457584007913129639935", // max uint256 = create ID 0
      "ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei", // metadata URI
      "1", // amount to mint
    ],
    contractAddress: process.env.ERC1155_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Mint initiated!");
  console.log(`   Transaction ID: ${mintResponse.data?.id}`);
  console.log(`\n⏳ WAIT for this mint to show COMPLETE before transferring!\n`);

  // BATCH TRANSFER: Send Token ID 0 to recipient
  console.log("📤 Batch transferring Token ID 0...");
  const transferResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "safeBatchTransferFrom(address,address,uint256[],uint256[],bytes)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      process.env.RECIPIENT_WALLET_ADDRESS!,
      ["0"], // token IDs to transfer
      ["1"], // amounts for each token ID
      "0x", // empty bytes (no extra data)
    ],
    contractAddress: process.env.ERC1155_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Transfer initiated!");
  console.log(`   Transaction ID: ${transferResponse.data?.id}`);
  console.log("\n🎉 Done! Check with npm run check-erc1155");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc1155
```

---

## Step 7: Execute an Airdrop (The Right Way)

The airdrop contract is a distribution machine. It needs three things:

1. **Tokens to distribute** — from your ERC-20 contract
2. **Permission to move them** — you must approve the airdrop contract first
3. **Enough balance** — you must own what you're sending

> **If your balance is less than the amount you want to airdrop, mint more first.**

### 7.1 Mint More ERC-20 Tokens (If Needed)

```bash
cat << 'EOF' > mint-erc20.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  console.log("🖨️  Minting 10 ERC-20 tokens...\n");

  const response = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "10000000000000000000", // 10 tokens (10 × 10^18)
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Mint initiated!");
  console.log(`   Transaction ID: ${response.data?.id}`);
  console.log("\n⏳ Wait for COMPLETE before doing anything else!");
}

main().catch(console.error);
EOF
```

```bash
npx tsx --env-file=.env mint-erc20.ts
```

### 7.2 Approve the Airdrop Contract

**Why do I need to approve?**
On the blockchain, no contract can move your tokens without your permission. The `approve` function says: "I allow the airdrop contract to spend up to X of my tokens." Without this step, the airdrop will silently fail.

```bash
cat << 'EOF' > approve-airdrop.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const tokenAddr = process.env.ERC20_CONTRACT_ADDRESS;
  const airdropAddr = process.env.AIRDROP_CONTRACT_ADDRESS;
  const walletId = process.env.WALLET_ID;

  if (!tokenAddr || tokenAddr.trim() === "" || tokenAddr.includes("PASTE")) {
    console.error("❌ ERC20_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  if (!airdropAddr || airdropAddr.trim() === "" || airdropAddr.includes("PASTE")) {
    console.error("❌ AIRDROP_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("🔐 Approving Airdrop contract to spend ERC-20 tokens...\n");
  console.log(`   Token Contract (ERC-20): ${tokenAddr}`);
  console.log(`   Spender (Airdrop):       ${airdropAddr}`);
  console.log(`   Approving Wallet:        ${walletId}\n`);

  const approveResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: walletId!,
    abiFunctionSignature: "approve(address,uint256)",
    abiParameters: [
      airdropAddr,
      "10000000000000000000", // Approve 10 tokens
    ],
    contractAddress: tokenAddr,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Approval initiated!");
  console.log(`   Transaction ID: ${approveResponse.data?.id}`);
  console.log("\n⏳ Wait for COMPLETE before running the airdrop!");
}

main().catch((err) => {
  console.error("\n❌ ERROR:", err.message);
  console.error("\n💡 Make sure ERC20_CONTRACT_ADDRESS and AIRDROP_CONTRACT_ADDRESS are set in .env");
});
EOF
```

Run:

```bash
npm run approve-airdrop
```

**Wait for COMPLETE before the next step.**

### 7.3 Execute the Airdrop

This script sends tokens to **two different recipients** — matching the official Arc doc pattern. This is how real airdrops work: one transaction, multiple recipients.

```bash
cat << 'EOF' > interact-airdrop.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const tokenAddr = process.env.ERC20_CONTRACT_ADDRESS;
  const airdropAddr = process.env.AIRDROP_CONTRACT_ADDRESS;
  const recipient = process.env.RECIPIENT_WALLET_ADDRESS;

  if (!tokenAddr || !airdropAddr || !recipient) {
    console.error("❌ Missing required addresses in .env");
    process.exit(1);
  }

  console.log("🚁 Executing ERC-20 Airdrop...\n");
  console.log(`   Token:     ${tokenAddr}`);
  console.log(`   Airdrop:   ${airdropAddr}`);
  console.log(`   Recipient: ${recipient}`);

  const recipients = [
    [recipient, "1000000000000000000"], // 1 token to your recipient
    [process.env.WALLET_ADDRESS!, "2000000000000000000"], // 2 tokens back to yourself
  ];

  console.log("\n📦 Sending:", JSON.stringify(recipients, null, 2));

  const response = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "airdropERC20(address,(address,uint256)[])",
    abiParameters: [tokenAddr, recipients],
    contractAddress: airdropAddr,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("\n✅ Airdrop initiated!");
  console.log(`   Transaction ID: ${response.data?.id}`);
  console.log("\n⚠️  Check this on the explorer. Look for:");
  console.log("   - 'Tokens Transferred' showing your ERC-20 moving");
  console.log("   - Not just USDC gas. If only USDC appears, it reverted.");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-airdrop
```

---

## Step 8: Verify Everything on the Explorer

After completing all the steps above, verify your work on the Arc Testnet Explorer.

### 8.1 Check Your Wallet

Open your browser and go to:
```
https://testnet.arcscan.app/address/YOUR_WALLET_ADDRESS
```

Replace `YOUR_WALLET_ADDRESS` with the value from your `.env` file.

You should see:
- Your token balances (ERC-20, ERC-721, ERC-1155)
- All your transaction history

### 8.2 Check a Specific Transaction

When you have a transaction hash (from `npm run check-erc20` etc.), visit:
```
https://testnet.arcscan.app/tx/YOUR_TRANSACTION_HASH
```

Look for:
- **Tokens Transferred** section — shows what moved
- **Status** — should say `Success`
- **From / To** — shows who sent and received

### 8.3 Check Circle API Logs

Go to your [Circle Console](https://console.circle.com) and check the API logs. You should see every request you made — deployments, mints, transfers, approvals, and airdrops.

---

## Summary

After completing this guide, you have:

- **Understood** how Circle and Arc work together (the signing flow, two types of IDs, transaction states)
- **Deployed** all four contract types (ERC-20, ERC-721, ERC-1155, Airdrop)
- **Tracked** every contract ID, transaction ID, and blockchain address in one `.env` file
- **Used** one universal `get-contract` script for all lookups
- **Minted** ERC-20 tokens (1 and 10 at a time)
- **Transferred** ERC-20 tokens to a recipient
- **Minted** an ERC-721 NFT and transferred it
- **Minted** an ERC-1155 token and batch-transferred it
- **Approved** the airdrop contract to spend your tokens
- **Executed** an airdrop to two different recipients
- **Verified** everything on the Arc Testnet Explorer

Your `.env` file is now a complete dashboard of your Arc Testnet empire — all managed from your phone.

**Next up:** [Guide 04 — Coming Soon](#)

Happy building!
