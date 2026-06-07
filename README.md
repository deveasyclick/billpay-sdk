# billpay

> Framework-agnostic Node.js SDK for bill payment processing in Nigeria — with multi-provider support, unified abstractions, and full TypeScript support.

[![npm version](https://img.shields.io/npm/v/billpay.svg)](https://www.npmjs.com/package/billpay)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![Node.js](https://img.shields.io/badge/Node.js->=16-green.svg)](https://nodejs.org/)

---

## Overview

**billpay** gives you a single, consistent interface to process utility bill payments in Nigeria across multiple providers. Point it at InterSwitch, VTPass, or both — and it will handle the rest.
- **Pay airtime, data, TV subscriptions, electricity, and gaming bills** through one unified API
- **Support for multiple providers** — InterSwitch and VTPass behind a single interface
- **No database required** — fully stateless; bring your own persistence
- **Drop into any framework** — Express, Fastify, NestJS, plain Node.js, whatever you use

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Initialization](#initialization)
  - [Multi-provider (with failover)](#multi-provider-with-failover)
  - [Single-provider clients](#single-provider-clients)
- [Core Concepts](#core-concepts)
  - [Provider preference](#provider-preference)
  - [Stateless architecture](#stateless-architecture)
- [API Reference](#api-reference)
  - [BillpayClient](#billpayclient)
  - [InterswitchClient](#interswitchclient)
  - [VtpassClient](#vtpassclient)
- [Common Workflows](#common-workflows)
  - [Airtime top-up](#airtime-top-up)
  - [Electricity payment](#electricity-payment)
  - [TV subscription](#tv-subscription)
  - [Transaction confirmation](#transaction-confirmation)
- [Configuration Reference](#configuration-reference)
- [Error Handling](#error-handling)
- [Environment Variables](#environment-variables)
- [Contributing](#contributing)
- [Changelog](#changelog)
- [License](#license)

---

## Installation

```bash
npm install billpay
# or
yarn add billpay
# or
pnpm add billpay
```

**Requirements:** Node.js ≥ 20, npm ≥ 10

---

## Quick Start

```ts
import { BillpayClient } from 'billpay';

const client = new BillpayClient({
  interswitch: {
    clientId: process.env.INTERSWITCH_CLIENT_ID!,
    secretKey: process.env.INTERSWITCH_SECRET_KEY!,
    terminalId: process.env.INTERSWITCH_TERMINAL_ID!,
    apiBaseUrl: 'https://sandbox.quickteller.com',
    authUrl: 'https://sandbox.quickteller.com/api/v5/Auth/GetAccessToken',
    paymentReferencePrefix: 'BPY_',
  },
  vtpass: {
    apiKey: process.env.VTPASS_API_KEY!,
    secretKey: process.env.VTPASS_SECRET_KEY!,
    publicKey: process.env.VTPASS_PUBLIC_KEY!,
    apiBaseUrl: 'https://sandbox.vtpass.com/api',
    phone: '08011111111',
  },
});

// Set active provider
client.setProviderPreference('INTERSWITCH');

// Browse available plans
const plans = await client.getPlans({ provider: 'BOTH' });

// Find the plan you need
const mtnPlan = plans.find(p => p.billerName === 'MTN' && p.amount === 50000);

// Pay
const result = await client.pay({
  billingItemId: mtnPlan.id,
  paymentReference: 'unique-ref-001',
  billerItem: mtnPlan,
  customerId: '08012345678',
  amount: 50000,
});

console.log(result);
```

---

## Initialization

### Multi-provider

Pass both `interswitch` and `vtpass` configurations to `BillpayClient` to use both providers:

```ts
import { BillpayClient } from 'billpay';

const client = new BillpayClient({
  interswitch: { /* InterSwitch config */ },
  vtpass:      { /* VTPass config */      },
});

client.setProviderPreference('INTERSWITCH');
```

When `pay()` is called, the SDK uses whichever provider is set as primary. Each provider is tried once; there is no automatic fallback between providers.

### Single-provider clients

If you only integrate one provider, import the dedicated client instead:

```ts
import { InterswitchClient from 'billpay/interswitch' };
import { VtpassClient }      from 'billpay/vtpass';

// InterSwitch only
const isClient = new InterswitchClient({
  interswitch: { /* config */ },
});

// VTPass only
const vtClient = new VtpassClient({
  vtpass: { /* config */ },
});
```

Single-provider clients expose the same full interface (`getPlans`, `pay`, `validateCustomer`, `confirmTransaction`, `getCategories`)

You can also achieve the same result with `BillpayClient` by supplying only one provider:

```ts
// Equivalent to InterswitchClient
const client = new BillpayClient({ interswitch: { /* config */ } });
```

---

## Core Concepts

### Provider preference

```ts
client.setProviderPreference('INTERSWITCH');
```

This sets InterSwitch as the active provider for all subsequent `pay()`, `getPlans()`, `validateCustomer()`, and `getCategories()` calls. To switch to VTPass:

```ts
client.setProviderPreference('VTPASS');
```

To use a specific provider for a single call without changing the global preference:

```ts
const result = await client.pay({
  ...paymentRequest,
  provider: 'VTPASS',
});
```

Check the current preference at any time:

```ts
const { primary, fallback } = client.getActiveProviders();
```

### Stateless architecture

The SDK holds **no persistent state** and has **no database dependency**. Every call is self-contained. This means:

- You own all transaction records (pending, failed, successful).

---

## API Reference

### `BillpayClient`

The main entry point. Accepts one or both provider configurations.

```ts
new BillpayClient(config: BillpayClientConfig)
```

#### `setProviderPreference(primary, fallback?)`

```ts
client.setProviderPreference('INTERSWITCH');
client.setProviderPreference('VTPASS');
```

#### `getActiveProviders()`

Returns `{ primary: ProviderType, fallback: ProviderType | null }`.

#### `getCategories(provider?)`

```ts
const categories = await client.getCategories('BOTH');
// => BillpayCategory[]
// e.g. [{ id: '1', name: 'Airtime' }, { id: '2', name: 'Data' }, ...]
```

Fetches unified bill categories (Airtime, Data, TV, Electricity, Gaming). When `'BOTH'` is specified, duplicates are removed and results are merged.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `provider` | `'INTERSWITCH' \| 'VTPASS' \| 'BOTH'` | current primary | Which provider to fetch categories from |

#### `getPlans(options?)`

```ts
const plans = await client.getPlans({
  provider: 'BOTH',
  filters: {
          vtpass: {
            "ELECTRICITY-BILL": ["Yola Electric Disco Payment - YEDC"],
          },
          interswitch: {
            "Cable TV Bills": ["DAARSAT Communications"],
          },
    },
});
// => BillerItem[]
//  [{
  //   category: 'Cable TV Bills',
  //   billerName: 'DAARSAT Communications',
  //   name: 'Single Package',
  //   amount: 600000,
  //   amountType: 5,
  //   active: true,
  //   paymentCode: '11310',
  //   billerId: '113',
  //   provider: 'INTERSWITCH'
  // },
  // ...
  //]
```

Fetches available billing plans. Provider-specific filters are passed directly to the underlying provider API.

| Option | Type | Description |
|--------|------|-------------|
| `provider` | `ProviderType \| 'BOTH'` | Which provider to query |
| `filters.interswitch` | `object` | Raw InterSwitch filter params |
| `filters.vtpass` | `object` | Raw VTPass filter params |

#### `validateCustomer(request)`

Validates a customer identifier before processing payment. Use this for electricity meters, decoder smartcard numbers, etc.

```ts
const customer = await client.validateCustomer({
  customerId: '45300023208',   // meter number, smartcard, phone, etc.
  paymentCode: plan.paymentCode,
  provider: 'INTERSWITCH',     // optional; defaults to primary
});
// => Customer { name, address, ... }
```

#### `pay(request)`

Executes a bill payment using the configured provider.

```ts
const result = await client.pay({
  billingItemId:    mtnPlan.id,
  paymentReference: 'unique-ref-001',  // must be globally unique per transaction
  billerItem:       mtnPlan,
  customerId:       '08012345678',
  amount:           50000,             // in kobo (50,000 kobo = ₦500)
  provider:         'INTERSWITCH',     // optional; overrides preference & disables failover
});
// => PayResponse
```

> **Important:** `paymentReference` must be unique per transaction. Reusing a reference may cause your provider to reject or misroute the payment.

#### `confirmTransaction(reference, provider?)`

Requery the status of a previously executed transaction.

```ts
const status = await client.confirmTransaction('unique-ref-001');
// => PayResponse { status, ... }
```

If `provider` is omitted, the SDK queries the primary provider. Pass a specific provider if you know which one processed the original payment.

---

### `InterswitchClient`

```ts
import InterswitchClient from 'billpay/interswitch';

new InterswitchClient({ interswitch: InterswitchConfig })
```

Exposes the same interface as `BillpayClient`. The `provider` parameter on any method is ignored (always uses InterSwitch). Attempting to set a VTPass fallback has no effect.

---

### `VtpassClient`

```ts
import VtpassClient from 'billpay/vtpass';

new VtpassClient({ vtpass: VtpassConfig })
```

Same interface as `BillpayClient`. Always uses VTPass; provider overrides are ignored.

---

## Common Workflows

### Airtime top-up

```ts
// 1. Get all airtime plans
const plans = await client.getPlans({
  provider: 'BOTH',
  filters: { vtpass: { serviceID: ['mtn'] } },
});

// 2. Pick a plan
const plan = plans.find(p => p.billerName === 'MTN' && p.amount === 10000);

// 3. Pay (no customer validation required for airtime)
const result = await client.pay({
  billingItemId: plan.id,
  paymentReference: `AIRTIME-${Date.now()}`,
  billerItem: plan,
  customerId: '08012345678', // recipient phone number
  amount: plan.amount,
});
```

### Electricity payment

```ts
// 1. Get electricity plans
const plans = await client.getPlans({
  provider: 'INTERSWITCH',
  filters: { interswitch: { categoryId: ['4'] } }, // electricity category
});

const plan = plans.find(p => p.billerName.includes('EKEDC'));

// 2. Validate the meter number first
const customer = await client.validateCustomer({
  customerId: '45300023208',
  paymentCode: plan.paymentCode,
});
console.log(`Validated: ${customer.name} at ${customer.address}`);

// 3. Pay
const result = await client.pay({
  billingItemId: plan.id,
  paymentReference: `ELEC-${Date.now()}`,
  billerItem: plan,
  customerId: '45300023208',
  amount: 500000, // ₦5,000 in kobo
});
```

### TV subscription

```ts
const plans = await client.getPlans({
  provider: 'BOTH',
  filters: { vtpass: { serviceID: ['dstv'] } },
});

const plan = plans.find(p => p.name.includes('Compact'));

// Validate smartcard
const customer = await client.validateCustomer({
  customerId: '7042552048',
  paymentCode: plan.paymentCode,
});

const result = await client.pay({
  billingItemId: plan.id,
  paymentReference: `TV-${Date.now()}`,
  billerItem: plan,
  customerId: '7042552048',
  amount: plan.amount,
});
```

### Transaction confirmation

Always confirm after payment — especially in webhook-driven or async flows:

```ts
const status = await client.confirmTransaction('unique-ref-001');

if (status.responseCode === '00') {
  // Success — update your records
} else {
  // Handle failure or pending state
}
```

---

## Configuration Reference

### `InterSwitchConfig`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `clientId` | `string` | ✅ | Your InterSwitch client ID |
| `secretKey` | `string` | ✅ | Your InterSwitch secret key |
| `terminalId` | `string` | ✅ | Your terminal ID |
| `apiBaseUrl` | `string` | ✅ | API base URL (sandbox or production) |
| `authUrl` | `string` | ✅ | OAuth token URL |
| `paymentReferencePrefix` | `string` | ❌ | Prefix for auto-generated payment references |

### `VtpassConfig`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | `string` | ✅ | Your VTPass API key |
| `secretKey` | `string` | ✅ | Your VTPass secret key |
| `publicKey` | `string` | ✅ | Your VTPass public key |
| `apiBaseUrl` | `string` | ✅ | API base URL (sandbox or production) |
| `phone` | `string` | ✅ | Phone number associated with the account |

### Sandbox vs Production URLs

| Provider | Sandbox | Production |
|----------|---------|------------|
| InterSwitch | `https://sandbox.quickteller.com` | `https://api.quickteller.com` |
| VTPass | `https://sandbox.vtpass.com/api` | `https://vtpass.com/api` |

---

## Error Handling

The SDK throws standard JavaScript `Error` objects with descriptive messages.

```ts
try {
  const result = await client.pay(paymentRequest);
} catch (error) {
  if (error instanceof Error) {
    console.error('Payment failed:', error.message);
    // error.message describes which providers were attempted and why each failed
  }
}
```

Common error scenarios and what they mean:

| Scenario | Behaviour |
|----------|-----------|
| Provider call fails | Error thrown with the provider's error message |
| `provider` override targets unconfigured provider | Error thrown immediately before any network call |
| No providers configured | Error thrown at construction time |
| Invalid `paymentReference` reuse | Provider-level error surfaced as thrown Error |

---

## Environment Variables

Store credentials in environment variables and never commit them to source control.

```env
# InterSwitch
INTERSWITCH_CLIENT_ID=your_client_id
INTERSWITCH_SECRET_KEY=your_secret_key
INTERSWITCH_TERMINAL_ID=your_terminal_id
INTERSWITCH_MERCHANT_CODE=your_merchant_code

# VTPass
VTPASS_API_KEY=your_api_key
VTPASS_SECRET_KEY=your_secret_key
VTPASS_PUBLIC_KEY=your_public_key
```

Use `dotenv` (included as a dependency) or your runtime's native secret management:

```ts
import 'dotenv/config';
import { BillpayClient } from 'billpay';

const client = new BillpayClient({
  interswitch: {
    clientId:  process.env.INTERSWITCH_CLIENT_ID!,
    secretKey: process.env.INTERSWITCH_SECRET_KEY!,
    terminalId: process.env.INTERSWITCH_TERMINAL_ID!,
    apiBaseUrl: 'https://sandbox.quickteller.com',
    authUrl:    'https://sandbox.quickteller.com/api/v5/Auth/GetAccessToken',
  },
});
```

---

## Contributing

Contributions are welcome! To get started:

```bash
git clone https://github.com/deveasyclick/billpay-sdk.git
cd billpay-sdk
pnpm install
```

**Useful scripts:**

```bash
pnpm build        # Compile TypeScript
pnpm test         # Run tests (vitest)
pnpm test:watch   # Watch mode
pnpm lint         # ESLint
pnpm type-check   # tsc without emit
```

Please open an issue before submitting a PR for significant changes. Bug fixes and documentation improvements are always welcome without prior discussion.

---

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for a full version history.

**Latest: [0.1.0] — 2026-03-06** — Initial release with InterSwitch and VTPass support, unified category abstractions, and full TypeScript types.

---

## License

[MIT](./LICENSE) © [deveasyclick](https://github.com/deveasyclick)
