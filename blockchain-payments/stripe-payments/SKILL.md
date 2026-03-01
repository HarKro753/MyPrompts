---
name: stripe-payments
description: Integrate Stripe for payment processing, escrow-like flows, Connect for multi-party payouts, and webhook handling. Use when building products that collect money, split payments, or hold funds before release.
---

# Stripe Payments

Extracted from: `stripe/stripe-node` (3k⭐), `stripe-samples/accept-a-payment`, `stripe-samples/subscription-use-cases`

## Core Mental Model

Stripe is three things: a payment processor, a financial infrastructure platform, and the best-designed API in fintech. The key abstraction is the **PaymentIntent** — a record of your intention to collect money. It tracks the full lifecycle: created → requires_action → processing → succeeded/failed.

For multi-party money flows (marketplace, escrow-like): use **Stripe Connect**. Connect lets you create sub-accounts, route funds between them, and control when money moves — without holding funds illegally.

## Setup

```bash
npm install stripe
```

```typescript
import Stripe from 'stripe'
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
})
```

## Accept a Payment (PaymentIntent flow)

```typescript
// Backend: create PaymentIntent
const paymentIntent = await stripe.paymentIntents.create({
  amount: 2000,          // in cents — €20.00
  currency: 'eur',
  automatic_payment_methods: { enabled: true },
  metadata: { commitmentId: '123', userId: 'abc' },  // pass context
})
// Return client_secret to frontend

// Frontend: confirm payment
import { loadStripe } from '@stripe/stripe-js'
const stripe = await loadStripe(process.env.NEXT_PUBLIC_STRIPE_PK!)
const { error } = await stripe.confirmPayment({
  elements,
  confirmParams: { return_url: 'https://yourapp.com/confirm' },
})
```

## Escrow-Like Flow with Stripe (for MVP before blockchain)

Stripe doesn't have a native escrow product, but you can simulate it:

```typescript
// 1. Collect funds — create PaymentIntent with manual capture
const paymentIntent = await stripe.paymentIntents.create({
  amount: 2000,
  currency: 'eur',
  capture_method: 'manual',  // funds authorized but not captured yet
  metadata: { commitmentId: '123' },
})

// 2. Capture when commitment resolves (user won)
await stripe.paymentIntents.capture(paymentIntent.id)

// 3. Cancel if commitment fails (user lost — use transfer instead)
await stripe.paymentIntents.cancel(paymentIntent.id)

// Limitation: manual capture expires after 7 days
// For longer holds: capture immediately into a Stripe balance, payout manually
```

## Stripe Connect — Multi-Party Payouts

For distributing winnings to multiple friends:

```typescript
// Create connected accounts (each friend gets one)
const account = await stripe.accounts.create({
  type: 'express',
  country: 'DE',
  email: 'friend@example.com',
})
// Return onboarding link
const link = await stripe.accountLinks.create({
  account: account.id,
  refresh_url: 'https://yourapp.com/reauth',
  return_url: 'https://yourapp.com/connected',
  type: 'account_onboarding',
})

// Transfer to connected account
await stripe.transfers.create({
  amount: 1500,  // €15.00 in cents
  currency: 'eur',
  destination: connectedAccountId,
  transfer_group: 'commitment_123',  // group related transfers
})
```

## Webhooks — the Critical Part

Payment state changes are async. Always use webhooks — never trust frontend confirmation alone.

```typescript
// pages/api/webhooks/stripe.ts (Next.js)
import { buffer } from 'micro'

export const config = { api: { bodyParser: false } }

export default async function handler(req, res) {
  const sig = req.headers['stripe-signature']!
  const body = await buffer(req)

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`)
  }

  switch (event.type) {
    case 'payment_intent.succeeded':
      const pi = event.data.object as Stripe.PaymentIntent
      // Mark commitment as funded in your DB
      await markCommitmentFunded(pi.metadata.commitmentId)
      break

    case 'payment_intent.payment_failed':
      // Handle failure
      break
  }

  res.json({ received: true })
}
```

```bash
# Test webhooks locally
stripe listen --forward-to localhost:3000/api/webhooks/stripe
stripe trigger payment_intent.succeeded
```

## Pricing Reality for Small Amounts

EU card: 1.4% + €0.25 per transaction.

For a €20 stake:
- Fee: €0.28 + €0.25 = **€0.53** (2.65% of transaction)
- Your 4% protocol fee: €0.80
- Net after Stripe: **€0.27** per commitment

At high volume this is fine. At €5 stakes it's untenable — minimum viable stake is ~€15 to keep economics healthy with Stripe. Below that, move to blockchain L2 (~€0.05 per transaction).

## Key Patterns

### Idempotency keys — prevent double charges
```typescript
await stripe.paymentIntents.create(
  { amount: 2000, currency: 'eur' },
  { idempotencyKey: `commitment_${commitmentId}_${userId}` }
)
```

### Metadata — pass context through Stripe
```typescript
// Always pass your internal IDs — makes webhook handling trivial
metadata: {
  commitmentId: '123',
  userId: 'abc',
  groupId: 'xyz',
}
```

### Test cards
- `4242 4242 4242 4242` — always succeeds
- `4000 0000 0000 0002` — always declines
- `4000 0025 0000 3155` — requires 3D Secure authentication

## Environment Setup

```bash
# .env
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...  # from stripe listen output
```
