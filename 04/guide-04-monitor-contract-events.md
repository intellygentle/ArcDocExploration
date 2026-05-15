# Guide 04 — Monitor Contract Events on Arc Testnet (Termux)

**A direct continuation of [Guide 03](https://github.com/intellygentle/ArcDocExploration/tree/main/03).** This guide teaches you how to **monitor** your contracts in real-time — get notified every time a token is minted, transferred, or airdropped — all from your Android phone.

---

## What You Will Learn

By the end of this guide you will understand and do:

- What **contract events** are and why they matter
- How to set up a **webhook endpoint** to receive real-time notifications
- How to register your webhook in the **Circle Developer Console**
- How to create an **event monitor** that tracks Transfer events
- How to **receive webhook notifications** when tokens move
- How to **retrieve past event logs** via the API

---

## What Are Contract Events? (The Basics)

Every time something happens on a smart contract — a token is minted, transferred, burned, or approved — the contract **emits an event**. Think of events as the blockchain's way of keeping a logbook.

### Why Events Matter

| Use Case | How Events Help |
|----------|----------------|
| **Track token transfers** | Know who sent what to whom, and when |
| **Build notifications** | Alert users when they receive tokens |
| **Monitor activity** | Watch for suspicious or unexpected behavior |
| **Build dashboards** | Display real-time token movement data |
| **Trigger automation** | Automatically do something when an event occurs |

### The Transfer Event

The most common event in blockchain is `Transfer`. Every ERC-20, ERC-721, and ERC-1155 token emits this event when tokens move between addresses.

```
Transfer(from, to, amount)
```

- `from` — who sent the tokens (address)
- `to` — who received the tokens (address)
- `amount` — how many tokens moved (number)

When you mint tokens, the `from` address is `0x0000...0000` (the zero address), because new tokens are being created out of thin air.

### Two Ways to Monitor Events

| Method | How It Works | Best For |
|--------|-------------|----------|
| **Webhooks (Push)** | Circle sends a notification to your URL the moment an event happens | Real-time monitoring, production apps |
| **Polling (Pull)** | You ask Circle's API "give me recent events" whenever you want | Testing, looking up past events |

In this guide, you'll set up **both**.

---

## Prerequisites

You **must** have completed [Guide 03](https://github.com/intellygentle/ArcDocExploration/tree/main/03). You need:

- Your `hello-arc` project with all contracts deployed
- At least one completed transaction (mint, transfer, or airdrop)
- Your `.env` file with all credentials and contract addresses

---

## Step 1: Return to Your Environment

Open **Termux** and run:

```bash
proot-distro login ubuntu
cd ~/hello-arc
```

---

## Step 2: Add npm Scripts

Run these to add the monitoring scripts:

```bash
npm pkg set scripts.webhook="tsx webhook-receiver.ts"
npm pkg set scripts.import-contract="tsx --env-file=.env import-contract.ts"
npm pkg set scripts.create-monitor="tsx --env-file=.env create-monitor.ts"
npm pkg set scripts.get-event-logs="tsx --env-file=.env get-event-logs.ts"
```

---

## Step 3: Set Up a Webhook Endpoint

A webhook endpoint is a URL that receives data when something happens. You need one so Circle can send you event notifications.

### Option A: webhook.site (Easiest for Mobile)

This is the simplest option — no installation needed.

1. Open your browser and go to **https://webhook.site**
2. You'll see a unique URL like `https://webhook.site/12345678-abcd-1234-efgh-...`
3. **Copy this URL** — you'll need it in the next step
4. **Keep the tab open** — you'll see notifications appear here in real-time

> **Why webhook.site?**
> It gives you a free, instant URL that captures any data sent to it. You can see the raw JSON payloads immediately. Perfect for learning and testing.

### Option B: ngrok (For Local Development)

If you want to run your own server, use ngrok to expose a local port to the internet.

1. Install ngrok (inside Ubuntu):

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok-v3-stable-linux-arm64.tgz | tar -xz -C /usr/local/bin
```

2. Create the webhook receiver script:

```bash
cat << 'EOF' > webhook-receiver.ts
import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

app.post("/webhook", (req: Request, res: Response) => {
  console.log("🔔 Received webhook:");
  console.log(JSON.stringify(req.body, null, 2));

  // Parse the important fields
  const notification = req.body?.notification;
  if (notification) {
    console.log("\n📋 Event Details:");
    console.log(`   Event:    ${notification.eventSignature}`);
    console.log(`   Contract: ${notification.contractAddress}`);
    console.log(`   Tx Hash:  ${notification.txHash}`);
    console.log(`   Block:    ${notification.blockHeight}`);
  }

  res.status(200).json({ received: true });
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Webhook receiver listening on port ${PORT}`);
  console.log(`Endpoint: http://localhost:${PORT}/webhook`);
});
EOF
```

3. Install express:

```bash
npm install express
npm install -D @types/express
```

4. Start the webhook receiver:

```bash
npm run webhook
```

5. In a **new Termux session** (swipe from left edge → New Session), start ngrok:

```bash
proot-distro login ubuntu
cd ~/hello-arc
ngrok http 3000
```

6. Copy the HTTPS forwarding URL (like `https://abc123.ngrok-free.app/webhook`)

---

## Step 4: Register Your Webhook in Circle Console

Before Circle can send you notifications, you need to tell it where to send them.

1. Open your browser and go to **https://console.circle.com**
2. Log in with your Circle developer account
3. In the left sidebar, click **Webhooks**
4. Click **Add a webhook**
5. Paste your webhook URL (from Step 3)
6. Click **Create**

> **Important:** You must register the webhook BEFORE creating event monitors. If you create a monitor first, Circle won't know where to send notifications.

---

## Step 5: Create Event Monitor Scripts

### 5.1 Import Contract Script (Optional)

If you deployed your contracts using Circle's templates (like you did in Guide 03), your contracts are already in the Console. **Skip this step.**

If you have a contract deployed elsewhere that you want to monitor, use this script to import it:

```bash
cat << 'EOF' > import-contract.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const contractClient = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function importContract() {
  const contractAddress = process.env.ERC20_CONTRACT_ADDRESS;

  if (!contractAddress) {
    console.error("❌ ERC20_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("📥 Importing ERC-20 contract into Circle Console...\n");
  console.log(`   Address: ${contractAddress}`);
  console.log(`   Blockchain: ARC-TESTNET\n`);

  try {
    const response = await contractClient.importContract({
      blockchain: "ARC-TESTNET",
      address: contractAddress,
      name: "MyERC20Token",
    });

    console.log("✅ Contract imported!");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    if (error.message?.includes("already exists")) {
      console.log("ℹ️  Contract is already in the Console. You can proceed to create monitors.");
    } else {
      console.error("❌ Error:", error.message);
    }
  }
}

importContract();
EOF
```

### 5.2 Create Monitor Script

This script creates an event monitor that tracks `Transfer` events on your ERC-20 contract.

```bash
cat << 'EOF' > create-monitor.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const contractClient = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function createEventMonitor() {
  const contractAddress = process.env.ERC20_CONTRACT_ADDRESS;

  if (!contractAddress) {
    console.error("❌ ERC20_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("📡 Creating event monitor for Transfer events...\n");
  console.log(`   Contract: ${contractAddress}`);
  console.log(`   Event:    Transfer(address,address,uint256)`);
  console.log(`   Network:  ARC-TESTNET\n`);

  try {
    const response = await contractClient.createEventMonitor({
      blockchain: "ARC-TESTNET",
      contractAddress: contractAddress,
      eventSignature: "Transfer(address,address,uint256)",
    });

    console.log("✅ Event monitor created!");
    console.log(JSON.stringify(response.data, null, 2));
    console.log("\n💡 Save the monitor ID above — you'll need it to manage this monitor later.");
    console.log("   Now mint or transfer tokens, and check your webhook for notifications!");
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) {
      console.error("Details:", JSON.stringify(error.response.data, null, 2));
    }
  }
}

createEventMonitor();
EOF
```

### 5.3 Get Event Logs Script

This script retrieves past event logs from the API — useful for looking up events that already happened.

```bash
cat << 'EOF' > get-event-logs.ts
import { initiateSmartContractPlatformClient } from "@circle-fin/smart-contract-platform";

const contractClient = initiateSmartContractPlatformClient({
  apiKey: process.env.CIRCLE_API_KEY!,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET!,
});

async function getEventLogs() {
  const contractAddress = process.env.ERC20_CONTRACT_ADDRESS;

  if (!contractAddress) {
    console.error("❌ ERC20_CONTRACT_ADDRESS is missing in .env!");
    process.exit(1);
  }

  console.log("📋 Fetching recent event logs...\n");
  console.log(`   Contract: ${contractAddress}`);
  console.log(`   Network:  ARC-TESTNET`);
  console.log(`   Limit:    10 events\n`);

  try {
    const response = await contractClient.listEventLogs({
      contractAddress: contractAddress,
      blockchain: "ARC-TESTNET",
      pageSize: 10,
    });

    const logs = response.data?.eventLogs || [];

    if (logs.length === 0) {
      console.log("📭 No event logs found yet.");
      console.log("   Mint or transfer some tokens first, then try again.");
      return;
    }

    console.log(`✅ Found ${logs.length} event(s):\n`);

    for (const log of logs) {
      console.log("─".repeat(50));
      console.log(`  Event:    ${log.eventSignature}`);
      console.log(`  Tx Hash:  ${log.txHash}`);
      console.log(`  Block:    ${log.blockHeight}`);
      console.log(`  Time:     ${log.firstConfirmDate}`);
      console.log(`  Topics:   ${JSON.stringify(log.topics)}`);
      console.log(`  Data:     ${log.data}`);
    }

    console.log("─".repeat(50));
    console.log("\n💡 Full response:");
    console.log(JSON.stringify(response.data, null, 2));
  } catch (error: any) {
    console.error("❌ Error:", error.message);
    if (error.response?.data) {
      console.error("Details:", JSON.stringify(error.response.data, null, 2));
    }
  }
}

getEventLogs();
EOF
```

---

## Step 6: Run the Scripts

### 6.1 Create the Event Monitor

```bash
npm run create-monitor
```

**You will see output like:**

```json
{
  "eventMonitor": {
    "id": "019bf984-b4da-7026-a3d2-674ce371a933",
    "contractName": "MyTokenContract",
    "contractId": "019bf8be-7be5-7a3e-89cc-05bcd7413f20",
    "contractAddress": "0x0d05a94dbf235f...",
    "blockchain": "ARC-TESTNET",
    "eventSignature": "Transfer(address,address,uint256)",
    "eventSignatureHash": "0xddf252ad...",
    "isEnabled": true
  }
}
```

**Save the monitor ID** — you'll need it to manage the monitor later.

### 6.2 Generate Some Events

Now trigger some Transfer events so you can see them in your webhook:

```bash
npm run interact-erc20
```

This mints and transfers tokens — each action creates a Transfer event.

### 6.3 Check Your Webhook

If you used **webhook.site**: Go back to the tab you kept open. You should see new requests appearing with the event data.

<img width="1200" height="2670" alt="Screenshot_2026-05-15-13-36-29-289_com lemurbrowser exts" src="https://github.com/user-attachments/assets/17b5f64c-2d8b-41fe-8e1f-7dd15c5c225c" />


If you used **ngrok**: Check the terminal running `npm run webhook`. You should see the event details printed.

### 6.4 Retrieve Past Event Logs

```bash
npm run get-event-logs
```

This shows you all the events that have been recorded for your contract, even if you missed the webhook.

---

## Understanding the Webhook Payload

When an event happens, Circle sends a JSON payload to your webhook. Here's what the fields mean:

```json
{
  "notificationType": "contracts.eventLog",
  "notification": {
    "eventSignature": "Transfer(address,address,uint256)",
    "contractAddress": "0x0d05a94dbf235f...",
    "blockchain": "ARC-TESTNET",
    "txHash": "0xe15d6dbb50178f...",
    "blockHeight": 22807198,
    "topics": [
      "0xddf252ad...",  // event signature hash
      "0x0000...0000",  // from address (padded to 32 bytes)
      "0x0000...bcf83d3b"  // to address (padded to 32 bytes)
    ],
    "data": "0x0000...0de0b6b3a7640000"  // amount (hex encoded)
  }
}
```

### Key Fields Explained

| Field | What It Means |
|-------|--------------|
| `notificationType` | Always `"contracts.eventLog"` for event monitors |
| `eventSignature` | Which event was emitted (e.g., Transfer) |
| `contractAddress` | Which contract emitted the event |
| `txHash` | The transaction hash — look it up on the explorer |
| `blockHeight` | The block number where this happened |
| `topics[0]` | The event signature hash (Keccak256 of the event name) |
| `topics[1]` | The `from` address (who sent) |
| `topics[2]` | The `to` address (who received) |
| `data` | The `amount` in hex (convert to decimal to read it) |

### How to Read the `data` Field

The `data` field contains the token amount in hexadecimal. To convert:

```
0x0de0b6b3a7640000 = 1000000000000000000 (decimal) = 1 token (with 18 decimals)
```

You can use this formula in your code:
```javascript
const amount = parseInt("0de0b6b3a7640000", 16);
// Result: 1000000000000000000
```

---

## What You Can Monitor

You can create monitors for **any event** your contract emits. Here are common ones:

### ERC-20 Events

| Event | When It Fires |
|-------|--------------|
| `Transfer(address,address,uint256)` | Tokens are minted, transferred, or burned |
| `Approval(address,address,uint256)` | Someone approves a spender |

### ERC-721 Events

| Event | When It Fires |
|-------|--------------|
| `Transfer(address,address,uint256)` | NFT is minted, transferred, or burned |
| `Approval(address,address,uint256)` | Owner approves someone to transfer their NFT |
| `ApprovalForAll(address,address,bool)` | Owner approves an operator for all their NFTs |

### ERC-1155 Events

| Event | When It Fires |
|-------|--------------|
| `TransferSingle(address,address,address,uint256,uint256)` | Single token type transferred |
| `TransferBatch(address,address,address,uint256[],uint256[])` | Multiple token types transferred |
| `ApprovalForAll(address,address,bool)` | Owner approves an operator |

### Airdrop Events

The airdrop contract emits Transfer events for each token it distributes — so monitoring Transfer on your ERC-20 contract catches airdrops too.

---

## Managing Your Monitors

### List All Monitors

You can view all your monitors in the Circle Console:
1. Go to **https://console.circle.com**
2. Navigate to **Contracts** → **Monitoring**
3. <img width="1195" height="1590" alt="Screenshot_2026-05-15-15-43-44-006_com android chrome-edit" src="https://github.com/user-attachments/assets/d3c9cbb3-957a-4765-810b-665537333a17" />

4. See all active monitors and their status

### Verify Webhook Delivery

In the Console, you can check if webhooks are being delivered:
1. Go to **Contracts** → **Monitoring**
2. Click on a monitor
3. See delivery status and recent webhook attempts

### Delete a Monitor

If you no longer need a monitor, you can delete it via the API or Console.

---

## Summary

After completing this guide, you have:

- **Understood** what contract events are and why they matter
- **Set up** a webhook endpoint (webhook.site or ngrok)
- **Registered** your webhook in the Circle Developer Console
- **Created** an event monitor that tracks Transfer events
- **Received** real-time webhook notifications when tokens move
- **Retrieved** past event logs via the API
- **Learned** how to read webhook payloads and decode event data

Your contracts are now being watched. Every time a token moves, you'll know about it.

**Next up:** [Guide 05 — Coming Soon](#)

Happy monitoring!
