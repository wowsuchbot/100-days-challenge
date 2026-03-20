# Day 5 Plan: Farcaster Mini App + x402 Payments for suchbot Credits

## Goal

Add payment/credit system for `@suchbot generate [prompt]` using Farcaster mini app integrated with x402 payments. Users buy credits, each generation consumes credits.

---

## Overview

```
User: "@suchbot generate a cyberpunk cat"
       ↓
Webhook: Check user has credits
       ↓ (yes) → Process generate → Deduct 1 credit → Return image
       ↓ (no) → Reply: "Buy credits at bot.mxjxn.com/credits"

User visits mini app
       ↓
x402 Payment: Pay 0.01 ETH for 100 credits
       ↓
Webhook: Receive payment webhook → Add credits to FID
```

---

## Architecture

### 1. Credit Storage

**Location:** Store in webhook database (SQLite or JSON file)

**Schema:**
```javascript
{
  fid: 4905,              // Farcaster user ID
  credits: 50,            // Current balance
  totalPurchased: 100,     // Lifetime credits purchased
  totalUsed: 50,           // Lifetime credits used
  lastUpdated: timestamp     // For analytics
}
```

**Why in webhook DB:**
- Already has access to user FID from Farcaster casts
- Centralized data for generate endpoint
- Easy to query on each request

### 2. x402 Payment Integration

**Based on:** `/root/.openclaw/projects/old-workspace/x402-miniapp/README.md`

**Flow:**
1. User clicks "Buy 100 credits for 0.01 ETH" in mini app
2. x402 payment UI opens (iframe/embed)
3. User approves payment on-chain
4. x402 POST webhook to our server with transaction receipt
5. Server verifies transaction → Credits added to user's FID

**Payment Webhook:**
```javascript
POST /webhook/x402-payment
{
  fid: 4905,
  transactionHash: "0x...",
  amount: "0.01",
  currency: "ETH",
  creditsAdded: 100
}
```

### 3. Rate Limiting + Credits

**Based on:** `/root/.openclaw/shared/memory/topics/rate-limiting.md`

**New Logic:**
```javascript
// Existing rate limits still apply
// PLUS: Must have sufficient credits

const rateLimit = await checkRateLimit(fid);
const credits = await getCredits(fid);

if (!rateLimit.allowed) {
  return { error: "Rate limit exceeded" };
}

if (credits.credits < 1) {
  return {
    error: "No credits",
    message: "Buy credits at https://bot.mxjxn.com/credits"
  };
}

// Process generate
await deductCredits(fid, 1);
```

### 4. Farcaster Mini App

**Based on:** `/root/.openclaw/projects/old-workspace/openclaw-repo/docs/farcaster-miniapps-checklist.md`

**Location:** `bot.mxjxn.com/credits` or `bot.mxjxn.com/buy-credits`

**Manifest (`.well-known/farcaster.json`):**
```json
{
  "accountAssociation": { /* signed */ },
  "frame": {
    "version": "1",
    "name": "suchbot Credits",
    "iconUrl": "https://bot.mxjxn.com/icon.png",
    "homeUrl": "https://bot.mxjxn.com/credits",
    "imageUrl": "https://bot.mxjxn.com/og.png",
    "button": {
      "title": "Buy Credits",
      "action": {
        "type": "launch_frame",
        "name": "Purchase",
        "url": "https://bot.mxjxn.com/credits/purchase"
      }
    }
  }
}
```

**Mini App Features:**
- Show current credit balance (query webhook `/credits?fid=4905`)
- Buy credits: 100 for 0.01 ETH, 500 for 0.05 ETH
- Payment history (last 10 transactions)
- Link to suchbot glitch/generate docs

### 5. Generate Flow Update

**Current:** `@suchbot generate [prompt]` → Process immediately

**New:**
```javascript
POST /generate
{
  text: "@suchbot generate a cyberpunk cat"
}

// 1. Extract FID from cast
// 2. Check credits
// 3. If credits > 0: Process, deduct, queue, reply
// 4. If credits = 0: Reply with buy link
```

---

## Project Structure

### Frontend (Mini App) → **bot-website repo**
- Location: `/root/.openclaw/services/bot-website/src/pages/credits/`
- Purpose: Public UI (credit balance, buy credits, payment history)
- Tech: Astro pages (like rest of bot-website)
- Why: Already a website repo, unified deployment

### Backend (Credits + Payment) → **webhook repo**
- Location: `/root/.openclaw/workspace/api/`
- Purpose: Credit DB + x402 payment webhook + credits check in generate
- Files:
  - `credits.js` - Credit database operations
  - `x402-webhook.js` - Payment webhook handler
  - `/generate` endpoint already exists (add credit check)
- Why: API logic belongs in webhook repo, payment webhook needs to be there

### Integration Flow
```
bot.mxjxn.com/credits (Astro mini app)
    ↓ User: "Buy 100 credits"
    ↓ x402: Payment on-chain
    ↓ Webhook: POST /webhook/x402-payment (webhook repo)
    ↓ Credits DB: Add credits to FID
    ↓ User: "@suchbot generate cat"
    ↓ Webhook: GET /credits?fid=4905 (check balance)
    ↓ Webhook: If has credits → deduct + process generate
```

The webhook's `/generate` endpoint is already there — just needs to:
1. Check credits first
2. If no credits → reply with buy link
3. If has credits → proceed with existing queue

---

## Implementation Steps

### Step 1: Credit Database

**File:** `/root/.openclaw/workspace/api/credits.js`

```javascript
// SQLite or JSON file storage
class CreditsDB {
  constructor(path) {
    this.db = new sqlite3.Database(path);
    this.init();
  }

  init() {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS credits (
        fid INTEGER PRIMARY KEY,
        credits INTEGER NOT NULL DEFAULT 0,
        totalPurchased INTEGER NOT NULL DEFAULT 0,
        totalUsed INTEGER NOT NULL DEFAULT 0,
        lastUpdated INTEGER NOT NULL DEFAULT ${Date.now()}
      )
    `);
  }

  async getCredits(fid) {
    const row = await this.db.get(
      'SELECT * FROM credits WHERE fid = ?',
      [fid]
    );
    return row || { fid, credits: 0, totalPurchased: 0, totalUsed: 0, lastUpdated: Date.now() };
  }

  async deductCredits(fid, amount) {
    const current = await this.getCredits(fid);
    if (current.credits < amount) return false;

    await this.db.run(
      'UPDATE credits SET credits = credits - ?, totalUsed = totalUsed + ?, lastUpdated = ? WHERE fid = ?',
      [amount, amount, Date.now(), fid]
    );
    return true;
  }

  async addCredits(fid, amount) {
    const current = await this.getCredits(fid);
    await this.db.run(
      'UPDATE credits SET credits = credits + ?, totalPurchased = totalPurchased + ?, lastUpdated = ? WHERE fid = ?',
      [amount, amount, Date.now(), fid]
    );
  }
}
```

### Step 2: Payment Webhook Endpoint

**File:** `/root/.openclaw/workspace/api/x402-webhook.js`

```javascript
import express from 'express';
import { CreditsDB } from './credits.js';

const router = express.Router();
const x402Secret = process.env.X402_WEBHOOK_SECRET;

router.post('/webhook/x402-payment', express.json(), async (req, res) => {
  // Verify x402 signature
  const signature = req.headers['x-x402-signature'];
  if (!verifyX402Signature(req.body, signature)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const { fid, transactionHash, amount, creditsAdded } = req.body;
  // Add credits to user
  await credits.addCredits(fid, creditsAdded);

  console.log(`✅ Payment received: FID ${fid} added ${creditsAdded} credits (tx: ${transactionHash})`);
  res.json({ success: true });
});
```

### Step 3: Update Generate Endpoint

**File:** `/root/.openclaw/workspace/api/index.js`

```javascript
import { CreditsDB } from './credits.js';

app.post('/generate', async (req, res) => {
  const { text, authorFid, castHash } = req.body;

  // Check credits
  const userCredits = await credits.getCredits(authorFid);

  if (userCredits.credits < 1) {
    // No credits - reply with buy link
    const buyUrl = 'https://bot.mxjxn.com/credits';
    const replyText = `You're out of credits! Buy more at ${buyUrl}`;
    await fc_cast.sh --text "${replyText}" --parent ${castHash};
    return res.json({ error: 'no_credits', message: 'No credits remaining' });
  }

  // Has credits - proceed
  await credits.deductCredits(authorFid, 1);

  // Queue generate request (existing system)
  await enqueueGenerate(text, authorFid, castHash);
  res.json({ success: true, creditsRemaining: userCredits.credits - 1 });
});
```

### Step 4: Mini App Implementation

**Directory:** `/root/.openclaw/services/bot-website/src/pages/credits/`

**Files:**
- `index.astro` - Credit balance display
- `purchase.astro` - Payment page with x402 integration
- `manifest.json` - `.well-known/farcaster.json`

**Purchase Page Code:**
```jsx
import { sdk } from '@farcaster/miniapp-sdk';

const prices = [
  { credits: 100, price: '0.001 ETH' },
  { credits: 500, price: '0.005 ETH' },
  { credits: 1000, price: '0.01 ETH' }
];

<Layout>
  <div>
    <h1>Buy Credits</h1>
    {prices.map(p => (
      <button onClick={async () => {
        const payment = sdk.actions.pay({
          to: '0x...',  // suchbot wallet
          tokenAddress: '0x...',  // WETH on Base
          amount: p.price
        });
        await payment.send();
      }}>
        Buy {p.credits} credits ({p.price})
      </button>
    ))}
  </div>
</Layout>
```

### Step 5: Credits Query API

**Endpoint:** `GET /credits?fid=4905`

```javascript
router.get('/credits', async (req, res) => {
  const { fid } = req.query;
  const userCredits = await credits.getCredits(parseInt(fid));
  res.json(userCredits);
});
```

---

## Testing Plan

### Test 1: No Credits
```
1. Set user credits to 0
2. Cast: "@suchbot generate test"
3. Expect: Reply with "Buy credits at bot.mxjxn.com/credits"
```

### Test 2: Payment Webhook
```
1. Simulate x402 webhook POST
2. Verify credits added to user FID
3. Check database: credits increased, totalPurchased updated
```

### Test 3: Credit Deduction
```
1. Set user credits to 5
2. Cast: "@suchbot generate test"
3. Generate processes
4. Check database: credits = 4, totalUsed = +1
```

### Test 4: Rate Limits Still Apply
```
1. Set user credits to 100
2. Trigger 20 generate requests quickly
3. Expect: Rate limit error (regardless of credits)
4. Wait 1 hour
5. Expect: Requests allowed again
```

### Test 5: Mini App Display
```
1. Visit bot.mxjxn.com/credits
2. Check balance displays correctly
3. Click "Buy 100 credits"
4. Verify x402 payment flow
5. After payment, refresh page - credits increased
```

---

## Files to Create/Modify

### New Files
- `/root/.openclaw/workspace/api/credits.js` - Credit database operations
- `/root/.openclaw/workspace/api/x402-webhook.js` - Payment webhook handler
- `/root/.openclaw/services/bot-website/src/pages/credits/index.astro`
- `/root/.openclaw/services/bot-website/src/pages/credits/purchase.astro`
- `/root/.openclaw/services/bot-website/public/.well-known/farcaster.json`

### Modified Files
- `/root/.openclaw/workspace/api/index.js` - Add credit check to generate endpoint, add x402 webhook route
- `/root/.openclaw/services/bot-website/README.md` - Add credits mini app to project list
- `/root/.openclaw/services/SERVICES.md` - Document credits system

---

## Potential Issues & Solutions

### Issue: Cold Start (0 credits for new users)
**Solution:** Free trial credits
- Give 3 free credits on first cast
- Store in database as `firstCastBonus: true`

### Issue: Webhook Security
**Solution:** Verify x402 signature
- `X-X402-Signature` header
- Shared secret in environment variables
- Reject unsigned requests

### Issue: Race Conditions
**Solution:** Database transactions
- Use SQLite `BEGIN TRANSACTION` for deduct/add operations
- Prevent double-spending

---

## Day 5 Deliverables

✅ Credit database (`credits.js`)
✅ Payment webhook (`x402-webhook.js`)
✅ Updated generate endpoint with credit check
✅ Credits query API (`GET /credits`)
✅ Farcaster mini app at `/credits`
✅ x402 payment integration
✅ Updated 100-days README with Day 5 entry

---

**Total scope estimate:** 4-6 hours (credits DB + x402 webhook + generate endpoint updates + mini app)

**References:**
- x402-miniapp README: `/root/.openclaw/projects/old-workspace/x402-miniapp/README.md`
- Farcaster mini app checklist: `/root/.openclaw/projects/old-workspace/openclaw-repo/docs/farcaster-miniapps-checklist.md`
- Rate limiting: `/root/.openclaw/shared/memory/topics/rate-limiting.md`
