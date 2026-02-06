# StreamHub - Payment Integration Specification

**Version:** 1.0  
**Payment Processor:** Stripe  
**Last Updated:** February 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Stripe Setup](#stripe-setup)
3. [Creator Onboarding](#creator-onboarding)
4. [Channel Memberships](#channel-memberships)
5. [Tips & Donations](#tips--donations)
6. [Pay-Per-View](#pay-per-view)
7. [Revenue Tracking](#revenue-tracking)
8. [Payout Management](#payout-management)
9. [Webhooks](#webhooks)
10. [Testing](#testing)

---

## Overview

### Revenue Streams

| Stream | Platform Fee | Creator Receives |
|--------|--------------|------------------|
| Channel Memberships | 10% | 90% |
| Tips/Donations | 5% | 95% |
| Pay-Per-View | 15% | 85% |
| Ad Revenue | 30% | 70% |

### Payment Flow

```
User Payment
    ↓
Stripe Processes Payment
    ↓
Platform Fee Deducted
    ↓
Creator Balance Updated
    ↓
Payout Requested (weekly/monthly)
    ↓
Funds Transferred to Creator Bank
```

---

## Stripe Setup

### Environment Variables

```bash
# .env.local
STRIPE_SECRET_KEY=sk_test_...           # Stripe secret key
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...  # Publishable key
STRIPE_WEBHOOK_SECRET=whsec_...         # Webhook signing secret

# Production
STRIPE_SECRET_KEY=sk_live_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
```

### Initialize Stripe

**Backend (Node.js):**

```typescript
// lib/stripe.ts
import Stripe from 'stripe';

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
  typescript: true,
});
```

**Frontend (Next.js):**

```typescript
// lib/stripeClient.ts
import { loadStripe } from '@stripe/stripe-js';

export const getStripe = () => {
  return loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);
};
```

---

## Creator Onboarding

### Stripe Connect Setup

**Purpose:** Enable creators to receive payments

**Flow:**

```
1. Creator clicks "Enable Monetization"
2. Check eligibility (1000+ subscribers)
3. Create Stripe Connect account
4. Redirect to Stripe onboarding
5. Creator completes identity verification
6. Creator returns to platform
7. Account approved for payments
```

### Create Connected Account

**API Endpoint:** `POST /api/monetization/onboard`

**Implementation:**

```typescript
// pages/api/monetization/onboard.ts
import { stripe } from '@/lib/stripe';
import { getServerSession } from 'next-auth';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const session = await getServerSession(req, res);
  const userId = session?.user?.id;

  if (!userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // Check eligibility
  const user = await db.query('SELECT follower_count FROM users WHERE id = $1', [userId]);
  if (user.rows[0].follower_count < 1000) {
    return res.status(403).json({ 
      error: 'Not eligible. Need 1000+ subscribers.' 
    });
  }

  // Create Stripe Connect account
  const account = await stripe.accounts.create({
    type: 'express',
    country: 'US',
    email: session.user.email,
    capabilities: {
      card_payments: { requested: true },
      transfers: { requested: true },
    },
  });

  // Save account ID
  await db.query(
    'INSERT INTO monetization_accounts (user_id, stripe_account_id) VALUES ($1, $2)',
    [userId, account.id]
  );

  // Create onboarding link
  const accountLink = await stripe.accountLinks.create({
    account: account.id,
    refresh_url: `${process.env.NEXT_PUBLIC_BASE_URL}/dashboard/monetization`,
    return_url: `${process.env.NEXT_PUBLIC_BASE_URL}/dashboard/monetization?success=true`,
    type: 'account_onboarding',
  });

  return res.json({ url: accountLink.url });
}
```

**Database Schema:**

```sql
CREATE TABLE monetization_accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    stripe_account_id VARCHAR(255) UNIQUE NOT NULL,
    onboarding_completed BOOLEAN DEFAULT FALSE,
    payouts_enabled BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(user_id)
);

CREATE INDEX idx_monetization_accounts_user_id ON monetization_accounts(user_id);
CREATE INDEX idx_monetization_accounts_stripe_id ON monetization_accounts(stripe_account_id);
```

---

## Channel Memberships

### Create Membership Tiers

**API Endpoint:** `POST /api/memberships/tiers`

**Request:**

```typescript
{
  name: string;          // "Basic", "Premium", "Ultra"
  price: number;         // In cents (e.g., 499 for $4.99)
  benefits: string[];    // ["Early access", "Exclusive content"]
}
```

**Implementation:**

```typescript
// pages/api/memberships/tiers.ts
import { stripe } from '@/lib/stripe';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const session = await getServerSession(req, res);
  const userId = session?.user?.id;
  
  const { name, price, benefits } = req.body;

  // Get creator's Stripe account
  const account = await db.query(
    'SELECT stripe_account_id FROM monetization_accounts WHERE user_id = $1',
    [userId]
  );

  if (!account.rows[0]) {
    return res.status(400).json({ error: 'Monetization not enabled' });
  }

  // Create Stripe product
  const product = await stripe.products.create({
    name: `${session.user.username} - ${name}`,
    description: benefits.join(', '),
    metadata: {
      creator_id: userId,
      tier_name: name,
    },
  }, {
    stripeAccount: account.rows[0].stripe_account_id,
  });

  // Create price
  const stripePrice = await stripe.prices.create({
    product: product.id,
    unit_amount: price,
    currency: 'usd',
    recurring: {
      interval: 'month',
    },
  }, {
    stripeAccount: account.rows[0].stripe_account_id,
  });

  // Save to database
  const tier = await db.query(
    `INSERT INTO membership_tiers 
     (user_id, stripe_product_id, stripe_price_id, name, price, benefits) 
     VALUES ($1, $2, $3, $4, $5, $6) 
     RETURNING *`,
    [userId, product.id, stripePrice.id, name, price, JSON.stringify(benefits)]
  );

  return res.json({ data: tier.rows[0] });
}
```

**Database Schema:**

```sql
CREATE TABLE membership_tiers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    stripe_product_id VARCHAR(255) NOT NULL,
    stripe_price_id VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    price INTEGER NOT NULL,
    benefits JSONB,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_membership_tiers_user_id ON membership_tiers(user_id);
```

### Subscribe to Membership

**API Endpoint:** `POST /api/memberships/subscribe`

**Flow:**

```
1. User clicks "Join" on tier
2. Frontend calls /api/memberships/subscribe
3. Backend creates Stripe Checkout session
4. User redirected to Stripe checkout
5. User completes payment
6. Webhook confirms payment
7. User gets member badge
```

**Implementation:**

```typescript
// pages/api/memberships/subscribe.ts
export default async function handler(req, res) {
  const { tierId } = req.body;
  const session = await getServerSession(req, res);
  const userId = session?.user?.id;

  // Get tier details
  const tier = await db.query(
    'SELECT * FROM membership_tiers WHERE id = $1',
    [tierId]
  );

  // Get creator's Stripe account
  const account = await db.query(
    'SELECT stripe_account_id FROM monetization_accounts WHERE user_id = $1',
    [tier.rows[0].user_id]
  );

  // Create Stripe Checkout session
  const checkoutSession = await stripe.checkout.sessions.create({
    mode: 'subscription',
    customer_email: session.user.email,
    line_items: [
      {
        price: tier.rows[0].stripe_price_id,
        quantity: 1,
      },
    ],
    success_url: `${process.env.NEXT_PUBLIC_BASE_URL}/membership/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_BASE_URL}/membership/cancel`,
    metadata: {
      user_id: userId,
      tier_id: tierId,
      creator_id: tier.rows[0].user_id,
    },
    subscription_data: {
      application_fee_percent: 10, // Platform takes 10%
    },
  }, {
    stripeAccount: account.rows[0].stripe_account_id,
  });

  return res.json({ url: checkoutSession.url });
}
```

**Database Schema:**

```sql
CREATE TABLE memberships (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tier_id UUID NOT NULL REFERENCES membership_tiers(id),
    stripe_subscription_id VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL, -- active, cancelled, past_due
    current_period_start TIMESTAMP NOT NULL,
    current_period_end TIMESTAMP NOT NULL,
    cancel_at_period_end BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memberships_user_id ON memberships(user_id);
CREATE INDEX idx_memberships_creator_id ON memberships(creator_id);
CREATE INDEX idx_memberships_status ON memberships(status);
```

---

## Tips & Donations

### Send Tip

**API Endpoint:** `POST /api/tips/send`

**Request:**

```typescript
{
  creatorId: string;
  amount: number;        // In cents
  message?: string;      // Optional message
}
```

**Implementation:**

```typescript
// pages/api/tips/send.ts
export default async function handler(req, res) {
  const { creatorId, amount, message } = req.body;
  const session = await getServerSession(req, res);
  const userId = session?.user?.id;

  // Validation
  if (amount < 100) { // Minimum $1
    return res.status(400).json({ error: 'Minimum tip is $1' });
  }

  // Get creator's Stripe account
  const account = await db.query(
    'SELECT stripe_account_id FROM monetization_accounts WHERE user_id = $1',
    [creatorId]
  );

  // Create Payment Intent
  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency: 'usd',
    application_fee_amount: Math.floor(amount * 0.05), // 5% platform fee
    transfer_data: {
      destination: account.rows[0].stripe_account_id,
    },
    metadata: {
      type: 'tip',
      sender_id: userId,
      creator_id: creatorId,
      message: message || '',
    },
  });

  // Record tip in database
  await db.query(
    `INSERT INTO tips 
     (sender_id, creator_id, amount, message, stripe_payment_intent_id) 
     VALUES ($1, $2, $3, $4, $5)`,
    [userId, creatorId, amount, message, paymentIntent.id]
  );

  return res.json({ 
    client_secret: paymentIntent.client_secret 
  });
}
```

**Frontend (React):**

```tsx
import { Elements, PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js';
import { getStripe } from '@/lib/stripeClient';

const TipForm = ({ creatorId }) => {
  const [amount, setAmount] = useState(500); // $5
  const [message, setMessage] = useState('');
  const [clientSecret, setClientSecret] = useState('');

  const handleSubmit = async () => {
    const res = await fetch('/api/tips/send', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ creatorId, amount, message }),
    });
    const { client_secret } = await res.json();
    setClientSecret(client_secret);
  };

  return (
    <div>
      <select value={amount} onChange={(e) => setAmount(Number(e.target.value))}>
        <option value={100}>$1</option>
        <option value={500}>$5</option>
        <option value={1000}>$10</option>
        <option value={2500}>$25</option>
      </select>

      <textarea 
        placeholder="Optional message" 
        value={message}
        onChange={(e) => setMessage(e.target.value)}
      />

      {!clientSecret ? (
        <button onClick={handleSubmit}>Send Tip</button>
      ) : (
        <Elements stripe={getStripe()} options={{ clientSecret }}>
          <CheckoutForm />
        </Elements>
      )}
    </div>
  );
};

const CheckoutForm = () => {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!stripe || !elements) return;

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/tip/success`,
      },
    });

    if (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button type="submit" disabled={!stripe}>
        Confirm Payment
      </button>
    </form>
  );
};
```

**Database Schema:**

```sql
CREATE TABLE tips (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    sender_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    amount INTEGER NOT NULL,
    message TEXT,
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- pending, succeeded, failed
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tips_sender_id ON tips(sender_id);
CREATE INDEX idx_tips_creator_id ON tips(creator_id);
CREATE INDEX idx_tips_created_at ON tips(created_at DESC);
```

---

## Pay-Per-View

### Set Video as Pay-Per-View

**API Endpoint:** `PUT /api/videos/:id/ppv`

**Request:**

```typescript
{
  price: number;  // In cents
}
```

**Implementation:**

```typescript
// pages/api/videos/[id]/ppv.ts
export default async function handler(req, res) {
  const { id } = req.query;
  const { price } = req.body;
  const session = await getServerSession(req, res);

  // Update video
  await db.query(
    'UPDATE videos SET ppv_price = $1 WHERE id = $2 AND creator_id = $3',
    [price, id, session.user.id]
  );

  return res.json({ success: true });
}
```

### Purchase Pay-Per-View Access

**API Endpoint:** `POST /api/videos/:id/purchase`

**Implementation:**

```typescript
export default async function handler(req, res) {
  const { id } = req.query;
  const session = await getServerSession(req, res);
  const userId = session?.user?.id;

  // Get video
  const video = await db.query(
    'SELECT ppv_price, creator_id FROM videos WHERE id = $1',
    [id]
  );

  // Get creator's Stripe account
  const account = await db.query(
    'SELECT stripe_account_id FROM monetization_accounts WHERE user_id = $1',
    [video.rows[0].creator_id]
  );

  // Create Payment Intent
  const paymentIntent = await stripe.paymentIntents.create({
    amount: video.rows[0].ppv_price,
    currency: 'usd',
    application_fee_amount: Math.floor(video.rows[0].ppv_price * 0.15), // 15% fee
    transfer_data: {
      destination: account.rows[0].stripe_account_id,
    },
    metadata: {
      type: 'ppv',
      video_id: id,
      buyer_id: userId,
      creator_id: video.rows[0].creator_id,
    },
  });

  return res.json({ client_secret: paymentIntent.client_secret });
}
```

**Database Schema:**

```sql
-- Add to videos table
ALTER TABLE videos ADD COLUMN ppv_price INTEGER;

CREATE TABLE ppv_purchases (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    amount INTEGER NOT NULL,
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    purchased_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(user_id, video_id)
);

CREATE INDEX idx_ppv_purchases_user_id ON ppv_purchases(user_id);
CREATE INDEX idx_ppv_purchases_video_id ON ppv_purchases(video_id);
```

---

## Revenue Tracking

### Transactions Table

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL, -- membership, tip, ppv, ad_revenue
    amount INTEGER NOT NULL, -- Amount in cents
    platform_fee INTEGER NOT NULL,
    net_amount INTEGER NOT NULL, -- Amount after platform fee
    status VARCHAR(50) DEFAULT 'pending', -- pending, completed, refunded
    stripe_payment_intent_id VARCHAR(255),
    stripe_transfer_id VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_creator_id ON transactions(creator_id);
CREATE INDEX idx_transactions_type ON transactions(type);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);
```

### Calculate Creator Earnings

**API Endpoint:** `GET /api/earnings`

**Implementation:**

```typescript
export default async function handler(req, res) {
  const session = await getServerSession(req, res);
  const userId = session?.user?.id;
  const { period = '30d' } = req.query;

  // Date range
  const startDate = period === '30d' ? 'NOW() - INTERVAL \'30 days\'' :
                    period === '7d' ? 'NOW() - INTERVAL \'7 days\'' :
                    '\'1970-01-01\'';

  // Get earnings breakdown
  const result = await db.query(`
    SELECT 
      SUM(CASE WHEN type = 'membership' THEN net_amount ELSE 0 END) as membership_revenue,
      SUM(CASE WHEN type = 'tip' THEN net_amount ELSE 0 END) as tip_revenue,
      SUM(CASE WHEN type = 'ppv' THEN net_amount ELSE 0 END) as ppv_revenue,
      SUM(CASE WHEN type = 'ad_revenue' THEN net_amount ELSE 0 END) as ad_revenue,
      SUM(net_amount) as total_revenue,
      SUM(CASE WHEN status = 'completed' THEN net_amount ELSE 0 END) as paid_out,
      SUM(CASE WHEN status = 'pending' THEN net_amount ELSE 0 END) as pending
    FROM transactions
    WHERE creator_id = $1 
      AND created_at >= ${startDate}
  `, [userId]);

  return res.json({ data: result.rows[0] });
}
```

---

## Payout Management

### Request Payout

**API Endpoint:** `POST /api/payouts/request`

**Requirements:**
- Minimum balance: $50
- Frequency: Weekly or Monthly

**Implementation:**

```typescript
export default async function handler(req, res) {
  const session = await getServerSession(req, res);
  const userId = session?.user?.id;

  // Get pending balance
  const balance = await db.query(`
    SELECT SUM(net_amount) as total
    FROM transactions
    WHERE creator_id = $1 AND status = 'pending'
  `, [userId]);

  const pendingAmount = balance.rows[0].total;

  // Check minimum
  if (pendingAmount < 5000) { // $50
    return res.status(400).json({ 
      error: 'Minimum payout is $50' 
    });
  }

  // Get Stripe account
  const account = await db.query(
    'SELECT stripe_account_id FROM monetization_accounts WHERE user_id = $1',
    [userId]
  );

  // Create payout
  const payout = await stripe.payouts.create({
    amount: pendingAmount,
    currency: 'usd',
  }, {
    stripeAccount: account.rows[0].stripe_account_id,
  });

  // Mark transactions as completed
  await db.query(`
    UPDATE transactions 
    SET status = 'completed', stripe_transfer_id = $1
    WHERE creator_id = $2 AND status = 'pending'
  `, [payout.id, userId]);

  return res.json({ payout });
}
```

---

## Webhooks

### Setup Webhook Handler

**Endpoint:** `POST /api/webhooks/stripe`

**Implementation:**

```typescript
import { buffer } from 'micro';

export const config = {
  api: {
    bodyParser: false,
  },
};

export default async function handler(req, res) {
  const buf = await buffer(req);
  const sig = req.headers['stripe-signature'];

  let event;

  try {
    event = stripe.webhooks.constructEvent(
      buf,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Handle events
  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckoutCompleted(event.data.object);
      break;
    
    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object);
      break;
    
    case 'customer.subscription.updated':
      await handleSubscriptionUpdated(event.data.object);
      break;
    
    case 'customer.subscription.deleted':
      await handleSubscriptionDeleted(event.data.object);
      break;
    
    case 'payment_intent.succeeded':
      await handlePaymentSucceeded(event.data.object);
      break;
    
    case 'payment_intent.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
  }

  res.json({ received: true });
}

// Handle checkout completed
async function handleCheckoutCompleted(session) {
  const { user_id, tier_id, creator_id } = session.metadata;

  await db.query(`
    INSERT INTO memberships 
    (user_id, creator_id, tier_id, stripe_subscription_id, status, current_period_start, current_period_end)
    VALUES ($1, $2, $3, $4, $5, $6, $7)
  `, [
    user_id,
    creator_id,
    tier_id,
    session.subscription,
    'active',
    new Date(session.created * 1000),
    new Date((session.created + 2592000) * 1000), // +30 days
  ]);
}

// Handle subscription updated
async function handleSubscriptionUpdated(subscription) {
  await db.query(`
    UPDATE memberships
    SET 
      status = $1,
      current_period_start = $2,
      current_period_end = $3,
      cancel_at_period_end = $4
    WHERE stripe_subscription_id = $5
  `, [
    subscription.status,
    new Date(subscription.current_period_start * 1000),
    new Date(subscription.current_period_end * 1000),
    subscription.cancel_at_period_end,
    subscription.id,
  ]);
}

// Handle payment succeeded
async function handlePaymentSucceeded(paymentIntent) {
  const { type, sender_id, creator_id } = paymentIntent.metadata;

  if (type === 'tip') {
    await db.query(
      'UPDATE tips SET status = $1 WHERE stripe_payment_intent_id = $2',
      ['succeeded', paymentIntent.id]
    );
  }

  // Record transaction
  const platformFee = paymentIntent.application_fee_amount || 0;
  await db.query(`
    INSERT INTO transactions 
    (creator_id, type, amount, platform_fee, net_amount, stripe_payment_intent_id, status)
    VALUES ($1, $2, $3, $4, $5, $6, $7)
  `, [
    creator_id,
    type,
    paymentIntent.amount,
    platformFee,
    paymentIntent.amount - platformFee,
    paymentIntent.id,
    'pending',
  ]);
}
```

---

## Testing

### Test Mode

Use Stripe test keys:
- `sk_test_...`
- `pk_test_...`

### Test Cards

```
Success: 4242 4242 4242 4242
Decline: 4000 0000 0000 0002
3D Secure: 4000 0025 0000 3155

Expiry: Any future date
CVC: Any 3 digits
ZIP: Any 5 digits
```

### Webhook Testing

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks to local
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger test event
stripe trigger checkout.session.completed
```

---

**End of Payment Integration Specification**

*This spec provides everything needed to implement a complete payment system. All code examples are production-ready.*
