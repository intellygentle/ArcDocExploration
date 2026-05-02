### Learn Arc On the Go: Create Circle api key, Create SCA Accounts, Deploy Smart Contracts on Arc Testnet Using Only Your Android Phone

**Complete beginner-friendly guide** for deploying ERC-20, ERC-721, ERC-1155, and Airdrop contracts on **Arc Testnet** using **Termux + Ubuntu** on Android.  
No PC required. Uses official Circle SDK (developer-controlled SCA wallets + Smart Contract Platform).

**Why this works perfectly on mobile**:
- Everything runs inside a full Ubuntu environment in Termux.
- Circle handles RPC, gas sponsorship, and blockchain connection automatically.
- No manual RPC URL needed (explained below).

**Original official tutorial this guide follows**: [Link to doc](https://docs.arc.network/arc/tutorials/deploy-contracts)

---

## Prerequisites
- Android phone with at least 2 GB free storage
- Termux installed from **F-Droid** (not Google Play)
- Circle developer account (free)

---

## Step 1: Install Termux + Ubuntu + Node.js

Open **Termux** and run these commands one by one:
```bash
pkg update && pkg upgrade -y
pkg install proot-distro curl -y
proot-distro install ubuntu
proot-distro login ubuntu
apt update && apt upgrade -y
apt install curl git -y
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
```

---

## Step 2: Create Project & Install Packages

*If you started from the previous guide which means you have the "hello-arc" folder already, then start from the "cd ~/hello-arc" command but if not, start from the " mkdir ~/hello-arc" command*

```bash
mkdir ~/hello-arc
```
move into the directory
```
cd ~/hello-arc
npm init -y
npm pkg set type=module
```

# One-click scripts
```
npm pkg set scripts.create-wallet="tsx --env-file=.env create-wallet.ts"
npm pkg set scripts.deploy-erc20="tsx --env-file=.env deploy-erc20.ts"
npm pkg set scripts.deploy-erc721="tsx --env-file=.env deploy-erc721.ts"
npm pkg set scripts.deploy-erc1155="tsx --env-file=.env deploy-erc1155.ts"
npm pkg set scripts.deploy-airdrop="tsx --env-file=.env deploy-airdrop.ts"
npm pkg set scripts.check-transaction="tsx --env-file=.env check-transaction.ts"
npm pkg set scripts.get-contract="tsx --env-file=.env get-contract.ts"

npm install @circle-fin/developer-controlled-wallets @circle-fin/smart-contract-platform
npm install --save-dev tsx typescript @types/node
```

---

## Step 3: Create Circle API Key (Browser – Do this FIRST!)

1. Go to [https://console.circle.com/](https://console.circle.com/)
2. Sign up / Log in
3. Left sidebar → **Keys** → **Create a key** → **Standard Key**
4. Name: `My Mobile Arc Deployer`
5. Click **Create**
6. **COPY THE FULL API KEY** (it starts with `TEST_API_KEY:` and contains two colons)

---

## Step 4: Create `.env` File

```bash
cat << 'EOF' > .env
CIRCLE_API_KEY=PASTE_YOUR_FULL_API_KEY_HERE
CIRCLE_ENTITY_SECRET=will_be_filled_in_next_step
CIRCLE_WEB3_API_KEY=PASTE_YOUR_FULL_API_KEY_HERE
WALLET_ID=will_be_filled_after_wallet_creation
WALLET_ADDRESS=will_be_filled_after_wallet_creation
TRANSACTION_ID=will_be_filled_after_deploy
CONTRACT_ID=will_be_filled_after_deploy
EOF
```

Edit it:

```bash
nano .env
```

Replace the two `PASTE_YOUR_FULL_API_KEY_HERE` lines with your real key.  
**Save & exit**: `Ctrl+O` → Enter → `Ctrl+X`

> **❓ Why is there no Arc Testnet RPC URL in .env?**  
> Circle’s official SDK automatically connects to Arc Testnet when you use `blockchain: "ARC-TESTNET"`. Circle manages the RPC, gas fees (via Circle Gas Station), and everything else. No extra configuration needed!

---

## Step 5: Generate & Register Entity Secret

```bash
cat << 'EOF' > register-entity-secret.ts
import crypto from 'crypto';
import { registerEntitySecretCiphertext } from "@circle-fin/developer-controlled-wallets";

async function main() {
  const apiKey = process.env.CIRCLE_API_KEY;
  if (!apiKey) {
    console.error("❌ CIRCLE_API_KEY not found in .env file!");
    process.exit(1);
  }

  const entitySecret = crypto.randomBytes(32).toString('hex');
  
  console.log('\n🔑 === YOUR ENTITY SECRET ===');
  console.log(entitySecret);
  console.log('⚠️  COPY THIS EXACT 64-CHARACTER STRING NOW AND SAVE IT!\n');

  await registerEntitySecretCiphertext({ apiKey, entitySecret });
  console.log('✅ Entity Secret successfully registered with Circle!');
}

main().catch(console.error);
EOF
```

Run:

```bash
npx tsx --env-file=.env register-entity-secret.ts
```

Copy the printed Entity Secret → edit `.env` and replace the line → save.

---

## Step 6: Create Your SCA Wallet

```bash
cat << 'EOF' > create-wallet.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const walletSetResponse = await client.createWalletSet({ name: "Mobile Wallet Set" });
  const walletsResponse = await client.createWallets({
    blockchains: ["ARC-TESTNET"],
    count: 1,
    walletSetId: walletSetResponse.data?.walletSet?.id ?? "",
    accountType: "SCA",
  });

  const wallet = walletsResponse.data?.wallets?.[0];
  console.log("\n✅ WALLET CREATED SUCCESSFULLY!");
  console.log("📋 COPY THIS EXACTLY → WALLET_ID     =", wallet?.id);
  console.log("📋 COPY THIS EXACTLY → WALLET_ADDRESS =", wallet?.address);
  console.log("\n💡 Now update .env with these two values.");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run create-wallet
```

Copy the two values shown → update `.env` → save.

---

## Step 7: Fund Your Wallet

Open browser → [https://faucet.circle.com](https://faucet.circle.com)  
→ Select **Arc Testnet** → paste your **WALLET_ADDRESS** → Request test USDC.

---

## Step 8: Create Status Check Scripts (Run once)

```bash
cat << 'EOF' > check-transaction.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await client.getTransaction({ id: process.env.TRANSACTION_ID! });
  console.log("📊 Transaction Status:");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF

cat << 'EOF' > get-contract.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const client = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  const response = await client.getContract({ id: process.env.CONTRACT_ID! });
  console.log("📋 Full Contract Details (including final contract address):");
  console.log(JSON.stringify(response.data, null, 2));
}

main().catch(console.error);
EOF
```

---

## Step 9: Deploy Contracts (One at a time)

### 9.1 Deploy ERC-20 Token

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

Run:  
```bash
npm run deploy-erc20
```

**After deployment – Check status**  
1. Copy `transactionId` and `contractId` (or `contractIds[0]`) from output.  
2. `nano .env` → update `TRANSACTION_ID` and `CONTRACT_ID` → save.  
3. Check status (repeat every 10–30 seconds until `state: "COMPLETE"`):  
   ```bash
   npm run check-transaction
   ```  
4. When complete, get final contract address:  
   ```bash
   npm run get-contract
   ```

---

### 9.2 Deploy ERC-721 (NFT)

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

**After deployment** → Use the exact same 4-step status check as ERC-20 above.

---

### 9.3 Deploy ERC-1155

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

**After deployment** → Same 4-step status check.

---

### 9.4 Deploy Airdrop Contract

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

**After deployment** → Same 4-step status check.

---

## Troubleshooting

- **"cannot find target wallet"** → Run `npm run create-wallet` again, update `WALLET_ID` and `WALLET_ADDRESS` in `.env`, and fund the new wallet.
- **"malformed API key"** → Create a fresh Standard API key from Circle console.
- **"Missing script"** → Run the `npm pkg set` commands in Step 2 again.
- **Deployment stuck on PENDING** → Wait 10–30 seconds and re-run `npm run check-transaction`.

## How to Return Later

```bash
proot-distro login ubuntu
cd ~/hello-arc
```

---

**You now have a complete mobile deployment environment for Arc Testnet!**  
Deploy as many contracts as you want directly from your phone.

Made for total novices by **Intellygentle** — share freely!  
Tested and working as of May 2026.

Happy deploying! 🚀
