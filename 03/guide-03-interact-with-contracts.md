# Learn Arc On the Go: Redeploy, Track & Interact with Smart Contracts on Arc Testnet Using Only Your Android Phone

**Complete beginner-friendly guide** — a direct continuation of [Guide 02](https://github.com/intellygentle/ArcDocExploration/tree/main/02). This guide teaches you how to **redeploy** your contracts with a clean naming system, **interact** with them (mint, transfer), and **airdrop** tokens — all from your phone.

**What you will learn:**
- A clean `.env` filing system so you never confuse contract IDs again
- How to redeploy ERC-20, ERC-721, ERC-1155, and Airdrop contracts
- How to mint tokens and NFTs
- How to transfer tokens safely (including the ERC-721 SCA fix)
- How to approve and execute airdrops without silent failures
- How to check balances and allowances before spending

**Why this works perfectly on mobile:**
- Everything runs inside the same Ubuntu environment from Guide 02
- Circle handles gas, RPC, and blockchain connections automatically
- No PC required

---

## Prerequisites

You **must** have completed [Guide 02](https://github.com/intellygentle/ArcDocExploration/tree/main/02). You need:

- Android phone with Termux + Ubuntu set up
- Circle developer account and API key
- Your `hello-arc` project folder with packages installed
- Test USDC in your wallet (from [faucet.circle.com](https://faucet.circle.com))

---

## Step 1: Return to Your Environment

Open **Termux** and run:

```bash
proot-distro login ubuntu
cd ~/hello-arc
```

You are back where Guide 02 left you.



### 2.1 Create Your New `.env`

Run this block. It adds these variables to your `.env` file with **empty placeholders**. You will fill them in as you deploy.

```bash
cat << 'EOF' > .env

# ==========================================
RECIPIENT_WALLET_ADDRESS=PASTE_SECOND_WALLET_ADDRESS_HERE

# ==========================================
# ERC-20 TOKEN CONTRACT
# Fill these in after deploying your ERC-20
# ==========================================
ERC20_CONTRACT_ID=
ERC20_CONTRACT_ADDRESS=
ERC20_TRANSACTION_ID=

# ==========================================
# ERC-721 NFT CONTRACT
# Fill these in after deploying your ERC-721
# ==========================================
ERC721_CONTRACT_ID=
ERC721_CONTRACT_ADDRESS=
ERC721_TRANSACTION_ID=

# ==========================================
# ERC-1155 MULTI-TOKEN CONTRACT
# Fill these in after deploying your ERC-1155
# ==========================================
ERC1155_CONTRACT_ID=
ERC1155_CONTRACT_ADDRESS=
ERC1155_TRANSACTION_ID=

# ==========================================
# AIRDROP CONTRACT
# Fill these in after deploying your Airdrop
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

**Edit it now** to add your real API key, entity secret, wallet ID, and wallet address:

```bash
nano .env
```

**Save & exit:** `Ctrl+S` → & → `Ctrl+X`

> **🤔 Why so many empty lines?**
> Think of `.env` as a form. Right now you are printing blank forms. As you deploy each contract, you will "fill in the form" with real IDs and addresses. This prevents you from accidentally using the wrong contract.

---

## Step 3: Create the Get-Contract Script to Save your contract addresses into the env file


### 3.1 Create the Script

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

### 3.2 Add All npm Scripts

Run these **one by one**:

```bash
npm pkg set scripts.get-contract="tsx --env-file=.env get-contract.ts"
npm pkg set scripts.check-tx="tsx --env-file=.env check-transaction.ts"
npm pkg set scripts.deploy-erc20="tsx --env-file=.env deploy-erc20.ts"
npm pkg set scripts.deploy-erc721="tsx --env-file=.env deploy-erc721.ts"
npm pkg set scripts.deploy-erc1155="tsx --env-file=.env deploy-erc1155.ts"
npm pkg set scripts.deploy-airdrop="tsx --env-file=.env deploy-airdrop.ts"
npm pkg set scripts.interact-erc20="tsx --env-file=.env interact-erc20.ts"
npm pkg set scripts.interact-erc721="tsx --env-file=.env interact-erc721.ts"
npm pkg set scripts.interact-erc1155="tsx --env-file=.env interact-erc1155.ts"
npm pkg set scripts.approve-airdrop="tsx --env-file=.env approve-airdrop.ts"
npm pkg set scripts.interact-airdrop="tsx --env-file=.env interact-airdrop.ts"
```

---

## Step 4: Redeploy All Contracts (One by One)

For **each contract**, the workflow is identical:

1. **Run** the deploy script
2. **Copy** the `transactionId` and `contractId` from the output
3. **Paste** them into `.env` under the correct prefixed variables
4. **Check** the transaction status until it says `COMPLETE`
5. **Run** `get-contract` to retrieve the blockchain address
6. **Paste** the address into `.env`
7. Move to the next contract

---

### 4.1 Deploy ERC-20 Token

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

**Do this immediately:**

```bash
nano .env
```

Find the ERC-20 section and fill it in:

```bash
ERC20_CONTRACT_ID=019d...        # from contractIds[0]
ERC20_TRANSACTION_ID=019c...     # from transactionId
# Leave ERC20_CONTRACT_ADDRESS empty for now
```

**Save:** `Ctrl+O` → Enter → `Ctrl+X`

**Get the blockchain address:**

```bash
CONTRACT_TYPE=ERC20 npm run get-contract
```

You will see a line like:
```
ERC20_CONTRACT_ADDRESS=0x2811...
```

Copy that line into your `.env`.

**🎉 ERC-20 is fully registered!**

---

### 4.2 Deploy ERC-721 (NFT)

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
ERC721_CONTRACT_ID=...        # from contractIds[0]
ERC721_TRANSACTION_ID=...     # from transactionId
```


**Get Contract Address**

```bash
CONTRACT_TYPE=ERC721 npm run get-contract
```

Paste `ERC721_CONTRACT_ADDRESS=0x...` into `.env`.

---

### 4.3 Deploy ERC-1155 (Multi-Token)

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

```bash
ERC1155_CONTRACT_ID=...
ERC1155_TRANSACTION_ID=...
```


**Get address:**

```bash
CONTRACT_TYPE=ERC1155 npm run get-contract
```

Paste `ERC1155_CONTRACT_ADDRESS=0x...` into `.env`.

---

### 4.4 Deploy Airdrop Contract

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

```bash
AIRDROP_CONTRACT_ID=...
AIRDROP_TRANSACTION_ID=...
```

Check:

```bash
# Update TRANSACTION_ID in .env to your AIRDROP_TRANSACTION_ID
npm run check-tx
```

Get address:

```bash
CONTRACT_TYPE=AIRDROP npm run get-contract
```

Paste `AIRDROP_CONTRACT_ADDRESS=0x...` into `.env`.

---

## Step 5: Interact with Your Contracts

Now that every contract is registered in `.env`, the interaction scripts know exactly which house to visit.

### 5.1 ERC-20: Mint + Transfer

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

  console.log("🚀 ERC-20 Interaction: Mint + Transfer\n");

  console.log("🖨️  Minting 1 token to your wallet...");
  const mintResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "1000000000000000000",
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Mint initiated:", JSON.stringify(mintResponse.data, null, 2));
  console.log(`\n⏳ WAIT for this mint to show COMPLETE before transferring!`);
  console.log(`   Transaction ID: ${mintResponse.data?.id}\n`);

  console.log("📤 Transferring 1 token to recipient...");
  const transferResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "transfer(address,uint256)",
    abiParameters: [
      process.env.RECIPIENT_WALLET_ADDRESS!,
      "1000000000000000000",
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Transfer initiated:", JSON.stringify(transferResponse.data, null, 2));
  console.log(`   Transaction ID: ${transferResponse.data?.id}`);
  console.log("\n🎉 Done! Check both transactions with npm run check-tx");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc20
```

---

### 5.2 ERC-721: Mint + Transfer

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

  console.log("🚀 ERC-721 Interaction: Mint + transferFrom (safe for SCA wallets)\n");

  console.log("🎨 Minting NFT #1...");
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
  console.log("✅ Mint initiated:", JSON.stringify(mintResponse.data, null, 2));
  console.log(`\n⏳ WAIT for this mint to show COMPLETE before transferring!`);
  console.log(`   Transaction ID: ${mintResponse.data?.id}\n`);

  console.log("📤 Transferring NFT #1 to recipient...");
  const transferResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "transferFrom(address,address,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      process.env.RECIPIENT_WALLET_ADDRESS!,
      "1",
    ],
    contractAddress: process.env.ERC721_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Transfer initiated:", JSON.stringify(transferResponse.data, null, 2));
  console.log(`   Transaction ID: ${transferResponse.data?.id}`);
  console.log("\n🎉 Done!");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc721
```

---

### 5.3 ERC-1155: Mint + Batch Transfer

```bash
cat << 'EOF' > interact-erc1155.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";

const circleDeveloperSdk = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function main() {
  console.log("🚀 ERC-1155 Interaction: Mint + Batch Transfer\n");

  console.log("🖨️  Minting Token ID 0 (creating new token type)...");
  const mintResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "mintTo(address,uint256,string,uint256)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      "115792089237316195423570985008687907853269984665640564039457584007913129639935",
      "ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei",
      "1",
    ],
    contractAddress: process.env.ERC1155_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Mint initiated:", JSON.stringify(mintResponse.data, null, 2));
  console.log(`\n⏳ WAIT for this mint to show COMPLETE before transferring!\n`);

  console.log("📤 Batch transferring Token ID 0...");
  const transferResponse = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "safeBatchTransferFrom(address,address,uint256[],uint256[],bytes)",
    abiParameters: [
      process.env.WALLET_ADDRESS!,
      process.env.RECIPIENT_WALLET_ADDRESS!,
      ["0"],
      ["1"],
      "0x",
    ],
    contractAddress: process.env.ERC1155_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });
  console.log("✅ Transfer initiated:", JSON.stringify(transferResponse.data, null, 2));
  console.log("\n🎉 Done!");
}

main().catch(console.error);
EOF
```

Run:

```bash
npm run interact-erc1155
```

> **🤔 Why that crazy number for the first mint?**
> That is `max uint256` — the largest number the blockchain can hold. Passing it tells the ERC-1155 contract: "Create a brand new token ID." The contract assigns it ID `0`. For later mints, pass `0` and the contract creates IDs `1`, `2`, `3`, etc.

---

## Step 6: Execute an Airdrop (The Right Way)

The airdrop contract is a distribution machine. It needs:
1. **Tokens to distribute** (from your ERC-20 contract)
2. **Permission to move them** (approval)
3. **Enough balance** (you must own what you're sending)

> **If your balance is less than the amount you want to airdrop, mint more first.**

### 6.2 Mint More Tokens (If Needed)

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
      "10000000000000000000",
    ],
    contractAddress: process.env.ERC20_CONTRACT_ADDRESS!,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Mint initiated:", JSON.stringify(response.data, null, 2));
  console.log(`\n⏳ Transaction ID: ${response.data?.id}`);
  console.log("   Wait for COMPLETE before doing anything else!");
}

main().catch(console.error);
EOF
```

```bash
npx tsx --env-file=.env mint-erc20.ts
```

### 6.3 Approve the Airdrop Contract

This gives the airdrop contract permission to move your tokens.

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
      "10000000000000000000",
    ],
    contractAddress: tokenAddr,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("✅ Approval initiated!");
  console.log(JSON.stringify(approveResponse.data, null, 2));
  console.log(`\n⏳ Transaction ID: ${approveResponse.data?.id}`);
  console.log("   Wait for COMPLETE before running the airdrop!");
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

**Wait for it to show `COMPLETE` before the next step.**

### 6.4 Execute the Airdrop

This script sends **1 token** to one recipient

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
    [recipient, "1000000000000000000"],
  ];

  console.log("\n📦 Sending:", JSON.stringify(recipients, null, 2));

  const response = await circleDeveloperSdk.createContractExecutionTransaction({
    walletId: process.env.WALLET_ID!,
    abiFunctionSignature: "airdropERC20(address,(address,uint256)[])",
    abiParameters: [tokenAddr, recipients],
    contractAddress: airdropAddr,
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  console.log("\n✅ Airdrop initiated:", JSON.stringify(response.data, null, 2));
  console.log(`\n📝 Transaction ID: ${response.data?.id}`);
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




## Summary

After completing this guide, you have:

✅ **Redeployed** all four contract types with a clean naming system  
✅ **Tracked** every contract ID, transaction ID, and blockchain address in one `.env` file  
✅ **Used** one universal `get-contract` script for all lookups  
✅ **Minted and Transfered ERC-20 Tokens**  
✅ **Minted ERC1155 Tokens**
✅ **Created Airdrop Contract**
✅ **Approved and executed airdrop using a separate airdrop contract**



Your `.env` file is now a complete dashboard of your Arc Testnet empire — all managed from your phone.

Made for total novices by **Intellygentle** — share freely!

Tested and working as of May 2026.

Happy building! 🚀🚁
