# Guide 05 — Register Your First AI Agent on Arc Testnet (Termux)

**A direct continuation of [Guide 04](https://github.com/intellygentle/ArcDocExploration/tree/main/04).** This guide teaches you how to **register an AI agent onchain** using the ERC-8004 standard — creating an onchain identity, recording reputation, and verifying credentials, all from your Android phone using **Circle wallets**.

---

## What You Will Learn

By the end of this guide you will understand and do:

- What **ERC-8004** is and why AI agents need onchain identities
- Why you need **two separate wallets** (owner + validator)
- How to **create wallets** using the Circle SDK
- How to **register an AI agent identity** as an ERC-721 NFT
- How to **record reputation** for your agent
- How to **request and verify validation** onchain
- How to **check everything on the Arc Testnet Explorer**

---

## What Is ERC-8004? (The Basics)

**ERC-8004** is an Ethereum standard for AI agent identity, reputation, and validation on the blockchain. It lets you:

| Capability | What It Does |
|-----------|-------------|
| **Identity** | Register your AI agent as a unique onchain entity (minted as an ERC-721 NFT) |
| **Reputation** | Record feedback scores for your agent (e.g., "completed 95% of trades successfully") |
| **Validation** | Request third-party verification of your agent's credentials (e.g., KYC verified) |

### Why Onchain Identity Matters

Imagine you're building an AI trading bot. Users want to know:
- Is this agent legitimate? (Identity)
- Has it performed well historically? (Reputation)
- Has someone trustworthy verified it? (Validation)

ERC-8004 answers all three questions onchain — transparent, verifiable, and tamper-proof.

### The Three Registries

ERC-8004 uses three smart contracts working together:

| Contract | Purpose | What You Do With It |
|----------|---------|-------------------|
| **IdentityRegistry** | Stores agent identities as NFTs | Register your agent, get a unique agent ID |
| **ReputationRegistry** | Stores reputation feedback | Give your agent a reputation score |
| **ValidationRegistry** | Stores validation requests and responses | Request and verify third-party validation |

### Why Two Wallets?

This is important: **the agent owner cannot record reputation for their own agent.** This prevents self-dealing — you can't just give yourself a perfect score.

You need two separate wallets:

| Wallet | Role | What It Does |
|--------|------|-------------|
| **Owner Wallet** | Registers identity, requests validation | Creates the agent, asks for verification |
| **Validator Wallet** | Records reputation, responds to validation | Gives feedback, confirms credentials |

Think of it like a job reference: you (owner) list your skills, but your former employer (validator) confirms them.

---

## ERC-8004 Contract Addresses (Arc Testnet)

You'll need these addresses throughout the guide. **Copy them exactly:**

| Contract | Address |
|----------|---------|
| **IdentityRegistry** | `0x8004A818BFB912233c491871b3d84c89A494BD9e` |
| **ReputationRegistry** | `0x8004B663056A597Dffe9eCcC1965A193B7388713` |
| **ValidationRegistry** | `0x8004Cb1BF31DAf7788923b405b754f57acEB4272` |

---

## Prerequisites

Before starting, you need:

- Completed [Guide 01](https://github.com/intellygentle/ArcDocExploration/tree/main/01) through [Guide 04](https://github.com/intellygentle/ArcDocExploration/tree/main/04)
- **Node.js 20+** installed (check with `node --version`)
- A **Circle Developer Console** account with an API key (Standard Key) and registered Entity Secret

---

## Step 1: Add the Script to Your Project

We're adding a new script to the existing `hello-arc` project. Open **Termux** and run:

```bash
cd ~/hello-arc
```

Make sure you have the Circle SDK installed (you should already have it from earlier guides):
**Run**
```bash
npm install @circle-fin/developer-controlled-wallets viem
```

---

## Step 2: Configure Your Environment

Your `.env` file should already have your Circle credentials from earlier guides.

If these values are missing, add them:

```bash
cat << 'EOF' >> ~/hello-arc/.env
CIRCLE_API_KEY=YOUR_API_KEY
CIRCLE_ENTITY_SECRET=YOUR_ENTITY_SECRET
EOF
```

Replace:
- `YOUR_API_KEY` with your Circle API key from the [Circle Developer Console](https://console.circle.com/)
- `YOUR_ENTITY_SECRET` with the 64-character Entity Secret you registered earlier

> **Security Warning:** Never share your API key or Entity Secret. Never commit `.env` to GitHub.

---

## Step 3: Create the Script

Create the `register-agent.ts` file in your `hello-arc` project:

```bash
cat << 'EOF' > ~/hello-arc/register-agent.ts
import { initiateDeveloperControlledWalletsClient } from "@circle-fin/developer-controlled-wallets";
import {
  createPublicClient,
  http,
  parseAbiItem,
  getContract,
  keccak256,
  toHex,
} from "viem";
import { arcTestnet } from "viem/chains";

// --- Contract Addresses ---
const IDENTITY_REGISTRY = "0x8004A818BFB912233c491871b3d84c89A494BD9e";
const REPUTATION_REGISTRY = "0x8004B663056A597Dffe9eCcC1965A193B7388713";
const VALIDATION_REGISTRY = "0x8004Cb1BF31DAf7788923b405b754f57acEB4272";

// Placeholder IPFS URI (replace with your own after uploading agent metadata)
const METADATA_URI =
  process.env.METADATA_URI ||
  "ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei";

// --- Circle Client ---
const circleClient = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

// --- Public Client (for reading onchain data) ---
const publicClient = createPublicClient({
  chain: arcTestnet,
  transport: http(),
});

// Helper: poll Circle API until transaction is confirmed
async function waitForTransaction(txId: string, label: string) {
  process.stdout.write(`  Waiting for ${label}`);
  for (let i = 0; i < 30; i++) {
    await new Promise((r) => setTimeout(r, 2000));
    const { data } = await circleClient.getTransaction({ id: txId });
    if (data?.transaction?.state === "COMPLETE") {
      const txHash = data.transaction.txHash;
      console.log(`\n  Tx: https://testnet.arcscan.app/tx/${txHash}`);
      return txHash;
    }
    if (data?.transaction?.state === "FAILED") {
      throw new Error(`${label} failed onchain`);
    }
    process.stdout.write(".");
  }
  throw new Error(`${label} timed out`);
}

async function main() {
  console.log("\n=== ERC-8004 AI Agent Registration ===\n");

  // Step 1: Create wallets
  console.log("--- Step 1: Create Wallets ---");

  const walletSet = await circleClient.createWalletSet({
    name: "ERC8004 Agent Wallets",
  });

  const walletsResponse = await circleClient.createWallets({
    blockchains: ["ARC-TESTNET"],
    count: 2,
    walletSetId: walletSet.data?.walletSet?.id ?? "",
    accountType: "SCA",
  });

  const ownerWallet = walletsResponse.data?.wallets?.[0]!;
  const validatorWallet = walletsResponse.data?.wallets?.[1]!;

  console.log(`  Owner:     ${ownerWallet.address}`);
  console.log(`  Validator: ${validatorWallet.address}\n`);

  // Step 2: Register agent identity
  console.log("--- Step 2: Register Agent Identity ---");
  console.log(`  Metadata URI: ${METADATA_URI}\n`);

  const registerTx = await circleClient.createContractExecutionTransaction({
    walletAddress: ownerWallet.address!,
    blockchain: "ARC-TESTNET",
    contractAddress: IDENTITY_REGISTRY,
    abiFunctionSignature: "register(string)",
    abiParameters: [METADATA_URI],
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  await waitForTransaction(registerTx.data?.id!, "registration");

  // Step 3: Retrieve agent ID
  console.log("\n--- Step 3: Retrieve Agent ID ---");

  const latestBlock = await publicClient.getBlockNumber();
  const blockRange = 10000n;
  const fromBlock = latestBlock > blockRange ? latestBlock - blockRange : 0n;

  const transferLogs = await publicClient.getLogs({
    address: IDENTITY_REGISTRY,
    event: parseAbiItem(
      "event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)"
    ),
    args: { to: ownerWallet.address as `0x${string}` },
    fromBlock,
    toBlock: latestBlock,
  });

  if (transferLogs.length === 0) {
    throw new Error("No Transfer events found — registration may have failed");
  }

  const agentId = transferLogs[transferLogs.length - 1].args.tokenId!.toString();

  const identityContract = getContract({
    address: IDENTITY_REGISTRY,
    abi: [
      {
        name: "ownerOf",
        type: "function",
        stateMutability: "view",
        inputs: [{ name: "tokenId", type: "uint256" }],
        outputs: [{ name: "", type: "address" }],
      },
      {
        name: "tokenURI",
        type: "function",
        stateMutability: "view",
        inputs: [{ name: "tokenId", type: "uint256" }],
        outputs: [{ name: "", type: "string" }],
      },
    ],
    client: publicClient,
  });

  const owner = await identityContract.read.ownerOf([BigInt(agentId)]);
  const tokenURI = await identityContract.read.tokenURI([BigInt(agentId)]);

  console.log(`  Agent ID:     ${agentId}`);
  console.log(`  Owner:        ${owner}`);
  console.log(`  Metadata URI: ${tokenURI}\n`);

  // Step 4: Record reputation
  console.log("--- Step 4: Record Reputation ---");

  const tag = "successful_trade";
  const feedbackHash = keccak256(toHex(tag));

  console.log(`  Score:    95`);
  console.log(`  Type:     0 (positive)`);
  console.log(`  Tag:      ${tag}\n`);

  const reputationTx = await circleClient.createContractExecutionTransaction({
    walletAddress: validatorWallet.address!,
    blockchain: "ARC-TESTNET",
    contractAddress: REPUTATION_REGISTRY,
    abiFunctionSignature:
      "giveFeedback(uint256,int128,uint8,string,string,string,string,bytes32)",
    abiParameters: [agentId, "95", "0", tag, "", "", "", feedbackHash],
    fee: { type: "level", config: { feeLevel: "MEDIUM" } },
  });

  await waitForTransaction(reputationTx.data?.id!, "reputation");

  // Step 5: Request validation
  console.log("\n--- Step 5: Request Validation ---");

  const requestURI = "ipfs://bafkreiexamplevalidationrequest";
  const requestHash = keccak256(
    toHex(`kyc_verification_request_agent_${agentId}`)
  );

  console.log(`  Validator: ${validatorWallet.address}`);
  console.log(`  Agent ID:  ${agentId}\n`);

  const validationReqTx =
    await circleClient.createContractExecutionTransaction({
      walletAddress: ownerWallet.address!,
      blockchain: "ARC-TESTNET",
      contractAddress: VALIDATION_REGISTRY,
      abiFunctionSignature:
        "validationRequest(address,uint256,string,bytes32)",
      abiParameters: [
        validatorWallet.address!,
        agentId,
        requestURI,
        requestHash,
      ],
      fee: { type: "level", config: { feeLevel: "MEDIUM" } },
    });

  await waitForTransaction(validationReqTx.data?.id!, "validation request");

  // Step 6: Validator responds
  console.log("\n--- Step 6: Validator Responds ---");
  console.log(`  Response: 100 (passed)\n`);

  const validationResTx =
    await circleClient.createContractExecutionTransaction({
      walletAddress: validatorWallet.address!,
      blockchain: "ARC-TESTNET",
      contractAddress: VALIDATION_REGISTRY,
      abiFunctionSignature:
        "validationResponse(bytes32,uint8,string,bytes32,string)",
      abiParameters: [
        requestHash,
        "100",
        "",
        "0x" + "0".repeat(64),
        "kyc_verified",
      ],
      fee: { type: "level", config: { feeLevel: "MEDIUM" } },
    });

  await waitForTransaction(validationResTx.data?.id!, "validation response");

  // Step 7: Verify validation status
  console.log("\n--- Step 7: Verify Validation Status ---");

  const validationContract = getContract({
    address: VALIDATION_REGISTRY,
    abi: [
      {
        name: "getValidationStatus",
        type: "function",
        stateMutability: "view",
        inputs: [{ name: "requestHash", type: "bytes32" }],
        outputs: [
          { name: "validatorAddress", type: "address" },
          { name: "agentId", type: "uint256" },
          { name: "response", type: "uint8" },
          { name: "responseHash", type: "bytes32" },
          { name: "tag", type: "string" },
          { name: "lastUpdate", type: "uint256" },
        ],
      },
    ],
    client: publicClient,
  });

  const [valAddr, , valResponse, , valTag] =
    (await validationContract.read.getValidationStatus([
      requestHash,
    ])) as readonly [`0x${string}`, bigint, number, `0x${string}`, string, bigint];

  console.log(`  Validator: ${valAddr}`);
  console.log(`  Response:  ${valResponse} (100 = passed)`);
  console.log(`  Tag:       ${valTag}`);

  // Summary
  console.log("\n=== Registration Complete ===\n");
  console.log("Your AI agent now has:");
  console.log("  - Onchain identity (ERC-721 NFT)");
  console.log("  - Reputation feedback (score: 95)");
  console.log("  - Validation credentials (KYC verified)\n");
  console.log("View your agent on the explorer:");
  console.log(`  https://testnet.arcscan.app/address/${ownerWallet.address}`);
  console.log(`\nAgent ID (token): ${agentId}\n`);
}

main().catch((error) => {
  console.error("\nError:", error.message ?? error);
  process.exit(1);
});
EOF
```

---

## Step 4: Run the Script

```bash
cd ~/hello-arc
npx tsx --env-file=.env register-agent.ts
```

**You will see output like:**

```
=== ERC-8004 AI Agent Registration ===

--- Step 1: Create Wallets ---
  Owner:     0x1234...abcd
  Validator: 0x5678...efgh

--- Step 2: Register Agent Identity ---
  Metadata URI: ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei

  Waiting for registration....
  Tx: https://testnet.arcscan.app/tx/0xabc123...

--- Step 3: Retrieve Agent ID ---
  Agent ID:     1
  Owner:        0x1234...abcd
  Metadata URI: ipfs://bafkreibdi6623n3xpf7ymk62ckb4bo75o3qemwkpfvp5i25j66itxvsoei

--- Step 4: Record Reputation ---
  Score:    95
  Type:     0 (positive)
  Tag:      successful_trade

  Waiting for reputation....
  Tx: https://testnet.arcscan.app/tx/0xdef456...

--- Step 5: Request Validation ---
  Validator: 0x5678...efgh
  Agent ID:  1

  Waiting for validation request....
  Tx: https://testnet.arcscan.app/tx/0xghi789...

--- Step 6: Validator Responds ---
  Response: 100 (passed)

  Waiting for validation response....
  Tx: https://testnet.arcscan.app/tx/0xjkl012...

--- Step 7: Verify Validation Status ---
  Validator: 0x5678...efgh
  Response:  100 (100 = passed)
  Tag:       kyc_verified

=== Registration Complete ===

Your AI agent now has:
  - Onchain identity (ERC-721 NFT)
  - Reputation feedback (score: 95)
  - Validation credentials (KYC verified)

View your agent on the explorer:
  https://testnet.arcscan.app/address/0x1234...abcd

Agent ID (token): 1
```

---

## Step 5: Verify on the Explorer

Open your browser and go to:

```
https://testnet.arcscan.app/address/YOUR_OWNER_WALLET_ADDRESS
```

Replace `YOUR_OWNER_WALLET_ADDRESS` with the owner wallet address from the output.

You should see **four transactions**:
1. Identity registration (to IdentityRegistry)
2. Reputation feedback (to ReputationRegistry)
3. Validation request (to ValidationRegistry)
4. Validation response (to ValidationRegistry)

---

## Understanding What Just Happened

Here's the full flow broken down:

```
Circle SDK                          Onchain
    |                                  |
    |-- createWalletSet() ------------>|  (Wallet Set)
    |-- createWallets(2x ARC-TESTNET)->|  (Owner + Validator SCA wallets)
    |                                  |
    |-- createContractExecution ------>|  register("ipfs://...")
    |   (owner wallet)                 |  (IdentityRegistry)
    |<-- Agent ID (NFT token) ---------|
    |                                  |
    |-- createContractExecution ------>|  giveFeedback(95, "successful_trade")
    |   (validator wallet)             |  (ReputationRegistry)
    |                                  |
    |-- createContractExecution ------>|  validationRequest(validator, agentId, ...)
    |   (owner wallet)                 |  (ValidationRegistry)
    |                                  |
    |-- createContractExecution ------>|  validationResponse(requestHash, 100, ...)
    |   (validator wallet)             |  (ValidationRegistry)
    |                                  |
    |-- getLogs (viem, read-only) ---->|  Transfer events → agent ID
    |-- getContract (viem, read-only)->|  getValidationStatus → verified!
```

### Why Circle SDK + viem?

| Tool | What It Does | When We Use It |
|------|-------------|----------------|
| **Circle SDK** | Creates wallets, executes transactions, manages gas | All write operations (register, giveFeedback, validationRequest, validationResponse) |
| **viem** | Reads onchain data directly | Read-only operations (getLogs, getValidationStatus, ownerOf, tokenURI) |

Circle handles wallet creation, key management, and gas sponsorship. viem handles reading onchain state. Together they cover everything.

### The Score System

In this tutorial, we hardcoded `score: 95` for demonstration. In production, scores should be calculated dynamically. Examples:

| Agent Type | Score Calculation |
|-----------|------------------|
| **Trading bot** | Based on profit/loss ratio or slippage |
| **Loan agent** | Based on repayment history |
| **Service bot** | Based on customer ratings |

---

## Troubleshooting

### "CIRCLE_API_KEY not found"

Make sure your `.env` file has both `CIRCLE_API_KEY` and `CIRCLE_ENTITY_SECRET` set. Check with:

```bash
cat ~/hello-arc/.env
```

### "No Transfer events found"

The script searches the last 10,000 blocks. If the registration happened longer ago, the event may be out of range. Run the script again immediately after registration.

### "Execution reverted" on giveFeedback

Make sure the script is using the **validator** wallet to call `giveFeedback`. The owner wallet cannot record reputation for its own agent.

### Transaction keeps pending

Arc Testnet is usually fast, but occasionally slow. The `waitForTransaction` helper polls for up to 60 seconds. If it times out, check the [explorer](https://testnet.arcscan.app/) to see if it confirmed.

---

## Key Concepts Recap

| Concept | What You Learned |
|---------|-----------------|
| **ERC-8004** | Standard for AI agent identity, reputation, and validation onchain |
| **IdentityRegistry** | Contract that mints agent identities as ERC-721 NFTs |
| **ReputationRegistry** | Contract that stores reputation feedback from validators |
| **ValidationRegistry** | Contract that manages validation requests and responses |
| **Two-wallet model** | Owner registers, validator confirms — prevents self-dealing |
| **Circle SDK** | Handles wallet creation, transaction execution, and gas sponsorship |
| **SCA wallets** | Smart Contract Accounts created by Circle for gas abstraction |

---

## Summary

After completing this guide, you have:

- **Created** two SCA wallets using Circle SDK (owner + validator)
- **Registered** an AI agent identity onchain (ERC-721 NFT)
- **Retrieved** your unique agent ID from Transfer events
- **Recorded** reputation feedback from the validator wallet
- **Requested** and **verified** validation credentials onchain
- **Confirmed** everything on the Arc Testnet Explorer

Your AI agent now has a verifiable onchain identity with reputation and validation. This is the foundation for building trusted autonomous agents on Arc.

**Next up:** [Guide 06 — Coming Soon](#)

Happy building!
