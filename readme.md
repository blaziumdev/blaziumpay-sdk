# @blazium/sdk

Official Node.js SDK for BlaziumPay - Production-ready crypto payment infrastructure.

## Features

- 🔐 **Secure**: HMAC-SHA256 webhook verification with timing-safe comparison
- 💰 **Multi-Chain**: TON, Solana, Bitcoin support
- 🎯 **Reward Locking**: Lock rewards at payment creation (400 coins = exactly 400 coins)
- 🔄 **Idempotency**: Built-in duplicate payment prevention
- ⚡ **Real-time**: Long polling and webhook support
- 📊 **Balance Management**: Track earnings and withdrawals
- 🛡️ **Type-Safe**: Full TypeScript support with Zod validation
- 🚫 **No Database Required**: Works with any backend stack

## Installation

```bash
npm install @blazium/sdk dotenv
```

## Configuration

Create a `.env` file in your project root:

```env
BLAZIUM_API_KEY=bz_test_...
BLAZIUM_WEBHOOK_SECRET=whsec_...
```

## Quick Start

```typescript
import 'dotenv/config'; // Load environment variables
import { BlaziumPayClient, BlaziumEnvironment, BlaziumFiat } from '@blazium/sdk';

// Wrap in async function to allow await
(async () => {
  const client = new BlaziumPayClient({
    apiKey: process.env.BLAZIUM_API_KEY as string,
    webhookSecret: process.env.BLAZIUM_WEBHOOK_SECRET,
    environment: BlaziumEnvironment.PRODUCTION
  });

  // Create payment with locked reward
  const payment = await client.createPayment(
    {
      amount: 10.00,
      currency: BlaziumFiat.USD,
      description: 'Premium Pack',
      rewardAmount: 400,
      rewardCurrency: 'coins'
    },
    {
      idempotencyKey: 'order_12345'
    }
  );

  console.log('Checkout URL:', payment.checkoutUrl);
  console.log('Reward (LOCKED):', payment.rewardAmount); // Always 400
})();
```

## Core Concepts

### Reward Metadata (Developer Controlled)

The `rewardAmount` and `rewardCurrency` fields are **optional metadata** that you can use to track what you plan to give users. 
**BlaziumPay does NOT automatically grant rewards** - you must implement your own logic in webhook handlers.

```typescript
// Set reward metadata when creating payment
const payment = await client.createPayment({
  amount: 3.00,
  currency: 'USD',
  rewardAmount: 1000, // Metadata: 1000 gold to grant
  rewardCurrency: 'gold' // Metadata: Gold currency
});

// The rewardAmount is stored in the payment object
// You must implement your own logic to grant rewards when payment confirms
```

### Payment Lifecycle

```
CREATE → PENDING → CONFIRMED → [Your Custom Logic]
         ↓
       EXPIRED / FAILED / CANCELLED
```

**Important:** When a payment reaches `CONFIRMED` status, **you must implement your own webhook handler** to grant 
premium features, add currency, unlock content, or perform any other actions. BlaziumPay handles payment processing; 
you handle the business logic.

### Webhook Verification (Critical)

**Important:** Only verified webhooks receive events from BlaziumPay. You must verify your webhook endpoint in the dashboard before it will receive any events. Unverified webhooks will not receive any webhook events.

```typescript
import express from 'express';

app.post('/webhooks/blazium', express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf.toString('utf8');
  }
}), async (req, res) => {
  const signature = req.headers['x-blazium-signature'];
  
  // CRITICAL: Verify signature
  // Note: If webhookSecret is not configured, this will throw a ValidationError
  // with a helpful message explaining that webhooks must be verified first
  if (!client.verifyWebhookSignature(req.rawBody, signature)) {
    return res.status(401).send('Invalid signature');
  }
  
  const webhook = client.parseWebhook(req.rawBody, signature);
  
  if (webhook.event === 'payment.confirmed') {
    const payment = webhook.payment;
    const userId = payment.metadata.userId;
    
    // YOUR CUSTOM LOGIC HERE - You decide what happens:
    
    // Example: Grant premium features
    if (payment.rewardCurrency === 'premium') {
      await database.users.update(userId, { isPremium: true });
    }
    
    // Example: Add in-game currency
    if (payment.rewardCurrency === 'coins') {
      await database.users.incrementCoins(userId, payment.rewardAmount);
    }
    
    // You have full control - implement whatever logic you need!
  }
  
  res.status(200).send({ received: true });
});
```

**Webhook Verification Process:**
1. Create a webhook endpoint in the dashboard
2. Verify the endpoint ownership (BlaziumPay will send a challenge token)
3. Save the webhook secret securely (shown only once after verification)
4. Configure the secret in your SDK client
5. Only verified webhooks receive events - unverified webhooks are ignored by BlaziumPay

## API Reference

### Initialize Client

```typescript
const client = new BlaziumPayClient({
  apiKey: string,              // Required: Your API key
  webhookSecret?: string,      // Optional: For webhook verification
  baseUrl?: string,            // Optional: Custom API URL
  timeout?: number,            // Optional: Request timeout (default: 15000ms)
  environment?: 'production' | 'sandbox'
});
```

### Create Payment

```typescript
const payment = await client.createPayment(
  {
    amount: number,              // Amount in fiat currency
    currency: string,            // USD, EUR, TRY, etc.
    description?: string,        // Payment description
    metadata?: object,           // Custom data
    redirectUrl?: string,        // Success redirect
    cancelUrl?: string,          // Cancel redirect
    expiresIn?: number,          // Expiration in seconds (60-86400)
    rewardAmount?: number,       // LOCKED reward amount
    rewardCurrency?: string,     // Reward currency
    rewardData?: object          // Additional reward data
  },
  {
    idempotencyKey?: string      // Prevent duplicates
  }
);
```

### Get Payment

```typescript
const payment = await client.getPayment(paymentId);

console.log(payment.status);       // PENDING, CONFIRMED, etc.
console.log(payment.rewardAmount); // Locked reward
console.log(payment.txHash);       // Blockchain transaction
```

### Wait for Confirmation

```typescript
try {
  const confirmed = await client.waitForPayment(
    paymentId,
    300000,  // 5 minutes timeout
    3000     // Poll every 3 seconds
  );
  
  console.log('Payment confirmed!', confirmed.txHash);
} catch (error) {
  console.error('Payment timeout or failed');
}
```

### Balance & Withdrawals

```typescript
// Check merchant balance
const balance = await client.getBalance('TON');

console.log('Total Earned:', balance.totalEarned);
console.log('Available:', balance.availableBalance);
console.log('Pending:', balance.pendingBalance);

// Request withdrawal
const withdrawal = await client.requestWithdrawal({
  chain: 'TON',
  amount: 10,
  destinationAddress: 'YOUR_TON_ADDRESS'
});
```

### Webhook Verification

```typescript
// Verify signature (timing-safe)
const isValid = client.verifyWebhookSignature(rawBody, signature);

// Parse webhook
const webhook = client.parseWebhook(rawBody, signature);

console.log(webhook.event);   // payment.confirmed, etc.
console.log(webhook.payment); // Full payment object
```

### Utility Methods

```typescript
client.isPaid(payment);              // true if CONFIRMED
client.isPartiallyPaid(payment);     // true if underpaid
client.isExpired(payment);           // true if expired
client.isFinal(payment);             // true if no more updates
client.getPaymentProgress(payment);  // % of amount paid
client.formatAmount(1.5, 'TON');     // "1.5000 TON"
```

## Error Handling

```typescript
import {
  BlaziumError,
  AuthenticationError,
  ValidationError,
  NetworkError,
  TimeoutError,
  PaymentError
} from '@blazium/sdk';

try {
  const payment = await client.createPayment({ /* ... */ });
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error('Invalid API key');
  } else if (error instanceof ValidationError) {
    console.error('Invalid input:', error.details);
  } else if (error instanceof NetworkError) {
    console.error('Network error, retry...');
  }
}
```

## TypeScript Support

Full TypeScript support with type definitions:

```typescript
import type {
  Payment,
  PaymentStatus,
  CreatePaymentParams,
  WebhookPayload,
  MerchantBalance
} from '@blazium/sdk';
```

## Security Best Practices

1. **Never trust frontend signals** - Always verify payments server-side
2. **Verify webhook signatures** - Use `verifyWebhookSignature()` - CRITICAL for security
3. **Use idempotency keys** - Prevent duplicate payments
4. **Implement your own reward logic** - BlaziumPay does NOT automatically grant rewards. You must implement webhook handlers to grant premium features, add currency, or perform other actions
5. **Use rewardAmount as metadata** - Store what you promise users, but implement your own logic to grant it
6. **Store API keys securely** - Use environment variables
7. **Implement timeout handling** - Network issues happen
8. **Log webhook failures** - Monitor for issues
9. **Make webhook handlers idempotent** - Handle duplicate webhook deliveries gracefully

## Requirements

- Node.js >= 18
- No database required
- No Prisma required
- Works with any backend framework (Express, Fastify, NestJS, etc.)

## Support

- Documentation: https://docs.blaziumpay.com
- Issues: https://github.com/blaziumpay/sdk/issues
- Email: support@blaziumpay.com

## License

MIT © BlaziumPay
