# Unlock Encrypted Data with a Fiber Payment — Basic Quest

**CKBoost · Build on CKB: Campaign #8 · Quest 1 (Basic)**

A documented walkthrough of a pay-to-unlock flow on **CKB Testnet**, using two browser-based
[Fiber Network](https://fiber.world) nodes. A seller encrypts a report and issues an invoice;
a buyer pays **1 Testnet CKB**, and the payment itself reveals the key that unlocks the report.

| | |
|---|---|
| **Tutorial** | [Unlock Encrypted Data with a Fiber Payment](https://fiber.world/docs/build/encrypted-data-payment) |
| **Fiber version** | v0.10.1 |
| **Network** | CKB Testnet only |

---

## Contents
1. [Overview](#overview)
2. [How the flow works](#how-the-flow-works)
3. [Step-by-step walkthrough](#step-by-step-walkthrough)
4. [Run details](#run-details)
5. [Key concepts](#key-concepts)
6. [Experiments](#experiments)
7. [Issues and how I resolved them](#issues-and-how-i-resolved-them)
8. [Reflection](#reflection)
9. [Safety](#safety)

---

## Overview

The tutorial runs three parties:

| Node | Role | Where it runs |
|---|---|---|
| **Buyer A** | Pays the invoice and decrypts the report | My browser (Fiber WASM) |
| **Bottle B** | Public Testnet node that forwards payments | Remote |
| **Seller C** | Encrypts the report and creates the invoice | My browser (Fiber WASM) |

Browser nodes can start connections but cannot accept them, so Buyer A and Seller C each open a
channel with Bottle B, which routes the payment between them.

---

## How the flow works

The core idea: **the AES decryption key is also the invoice's payment preimage.**
Seller C cannot collect the payment without revealing that secret, so paying and unlocking
happen in the same moment, with no backend and no trusted middleman.

---

## Step-by-step walkthrough

### Step 1 - Setup: start nodes and open channels
- Started **Buyer A** and **Seller C** in the browser (Fiber WASM runtime, IndexedDB storage, WSS transport).
- Funded each node's own Testnet address from the CKB Testnet Faucet.
- Opened a channel from each node to **Bottle B** and waited for both to reach `CHANNEL_READY`.

Each channel moves through six opening states:
`NEGOTIATING_FUNDING` → `COLLABORATING_FUNDING_TX` → `SIGNING_COMMITMENT` →
`AWAITING_TX_SIGNATURES` → `AWAITING_CHANNEL_READY` → `CHANNEL_READY`

![Both channels at CHANNEL_READY](<screenshots/Screenshot (373).png>)
*Both channels A–B and C–B showing `CHANNEL_READY`.*

### Step 2 - Encrypt and create the invoice
- As Seller C, entered my own sample text:
  > The CKB tokens are all in my Binance wallet. I will be the winner of the CKB Campaign 8
- The browser generated a fresh 256-bit AES key and a random 96-bit IV, then encrypted the text with **AES-256-GCM**.
- Created a **1 CKB invoice** using the AES key as the payment preimage. The invoice's payment hash is the SHA-256 fingerprint of that key.

### Step 3 - Transfer the offer and try to decrypt without the key
- The encrypted offer was delivered to Buyer A as JSON containing `cipher`, `ciphertext`, `iv`, `invoice`, `paymentHash` and `lastHop`.
- Selected **Decrypt without key**. AES-GCM rejected the attempt: the buyer has the ciphertext and IV, but not Seller C's original key.

![Encrypted offer and decryption rejected](<screenshots/Screenshot (392).png>)
*The encrypted offer on the buyer side and the "Decryption rejected" message.*

### Step 4 - Pay the invoice
- Before sending funds, Buyer A automatically checked the invoice's **network, amount and payment hash** against the offer.
- Paid 1 Testnet CKB through Bottle B.

![Payment receipt](<screenshots/Screenshot (395).png>)
*Completed payment: 1 CKB received by Seller C, with the payment hash.*

### Step 5 - Decrypt the report
- Seller C revealed the preimage to claim the payment, and Fiber returned it to Buyer A.
- Selected **Decrypt result**: the preimage worked as the AES key and revealed the original text.

![Decrypted result](<screenshots/Screenshot (397).png>)
*The buyer's decrypted result matches the seller's original sample text.*

---

## Run details

| Field | Value |
|---|---|
| Cipher | AES-256-GCM |
| IV | `0x41e0549d403e442847e9da17` |
| Payment hash | `0x5a1602091c29124a5b48298136cbf88eee8d82578b07f96a999fde64dea9ed7e` |
| Invoice amount | 1 CKB |

### Channel economics
| | Buyer A / Seller C | Bottle B |
|---|---|---|
| Contributed per channel | 500 CKB | 250 CKB |
| Reserved for on-chain settlement | 100 CKB | 100 CKB |
| Spendable | 400 CKB | 150 CKB |

Because Bottle forwards from its own balance on B–C, the route from A to C is limited to
**150 CKB before fees**. That is plenty for a 1 CKB invoice.

---

## Key concepts

- **Payment preimage as decryption key.** The same secret settles the payment and unlocks the data, so the buyer receives the key exactly when the seller gets paid.
- **Payment hash as fingerprint.** The invoice shows only the SHA-256 hash of the key. The buyer can check the revealed key against it after payment.
- **Authenticated encryption.** AES-GCM both hides the content and detects any change to it. A wrong key or altered ciphertext is refused outright rather than producing garbled text.
- **Inbound vs outbound liquidity.** Outbound is what you can send from your side of a channel; inbound is what you can receive. Seller C's inbound liquidity is Bottle's balance on B–C.
- **What payment does *not* guarantee.** Settlement proves the released key matches the ciphertext. It does not prove the report is accurate or useful. Real marketplaces need previews, seller reputation or dispute rules for that.

---

## Experiments

### Optional liquidity transfer
Sent 5 CKB from Seller C to Bottle B to watch the B–C balance bar shift. No new CKB is created;
the same channel total simply moves from one side to the other.

### Integrity protection
AES-GCM is authenticated encryption: alongside the ciphertext it carries an authentication
tag that is checked before decryption. If even one character of the ciphertext were changed,
the check would fail and decryption would be refused rather than producing garbled text.
This protects the buyer from receiving altered content.

---

## Issues and how I resolved them

| # | Step | Expected | What happened | Resolution |
|---|---|---|---|---|
| 1 | Setup: channel liquidity | A clear sign of whether each step was required to complete the purchase | The page asked me to send 5 CKB from Seller C to Bottle B, but I didn't understand what the transfer was for or whether I needed it before paying | Re-read the liquidity section and worked out that it's an optional demo. It shows the B–C balance shifting between the two sides of the channel. Bottle already had 150 CKB on its side, far more than the 1 CKB invoice needed. I sent the 5 CKB anyway to watch the bar move. |
| 2 | Setup: optional 5 CKB transfer (Seller C → Bottle B) | The transfer to finish almost instantly, since payments inside an open channel don't need a new on-chain transaction | The button stayed on "Transferring…" for about 3 minutes, and the balance bar didn't move, so I wasn't sure whether it was stuck or still processing | Waited without clicking again, to avoid sending a duplicate transfer. It completed and showed "5 CKB sent", with the B–C bar updating to the new balances. |

---

## Reflection

**The format**
This was not my first Fiber tutorial. Having the explanation next to each step felt natural, because I never had to scroll away to understand what I had just clicked. It helped most when I opened the channels. I could watch each state tick off, from NEGOTIATING_FUNDING to CHANNEL_READY, while the text explained what was happening.

**What surprised me**
What stood out most was that the AES key and the payment preimage are the same thing. I didn't expect paying an invoice to also be how the key gets delivered. It made me realise the seller can't collect the money without revealing the key, so the buyer doesn't have to trust anyone to send it afterwards.

**Where I was unsure**
At the liquidity step, I wasn't sure whether the 5 CKB transfer from Seller C to Bottle B was required. My first thought was that the payment would fail if I skipped it. After reading the route numbers, I understood that 150 CKB of inbound liquidity was far more than a 1 CKB payment needs. The transfer was there to show the balances moving, and I watched B–C change to 153 / 397.

**A problem I ran into**
When I sent the 5 CKB, I expected it to be instant. Instead, the button stayed on "Transferring…" for about 3 minutes. I waited instead of clicking again, because I worried a second click would send another 5 CKB, and it completed.

**What I'd build**
I'd like to build a pay-to-unlock page for tutors in Lagos who share past exam questions. Right now, once a tutor sends the file to one student, it spreads in group chats and the tutor isn't paid. With pay-to-unlock, each student pays a small amount and gets the key instantly, and the tutor doesn't have to chase anyone for payment.

**One thing I'd change**
If I could change one thing, I would keep the finished run visible after clicking "Start another offer." The stepper resets and Steps 3–5 lose their checkmarks, which made me think my completed purchase had been undone. A small "Completed" badge or the previous receipt would make this clear.

---

## Safety

- Testnet CKB only. No Mainnet funds were used at any point.
- No private keys, seed phrases, passwords or browser-storage contents are included in this repository or its screenshots.
- All values shown (IV, payment hash, invoice) are public Testnet data.

---

## Links
- Tutorial: [Unlock Encrypted Data with a Fiber Payment](https://fiber.world/docs/build/encrypted-data-payment)
- Campaign: [CKBoost — Build on CKB: Campaign #8](https://ckboost.netlify.app/campaign/0x0846b87630f0c8cf2e4179bed0cf2b0dd0ed438b203a4be1129024465049cda9)
- Fiber Network docs: [fiber.world](https://fiber.world)
