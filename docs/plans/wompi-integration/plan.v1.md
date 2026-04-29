---
model: various
code_agent: Claude Code / OpenCode
created_at: 2026-04-23
updated_at: 2026-04-28
status: in_progress
---

# Wompi Payment Provider Implementation Plan

## Context

This plan outlines the implementation of a Wompi payment provider plugin for Medusa v2. The plugin is created in `medusa-wompi/` directory and can be installed in different Medusa projects.

**Why this change:** Caribestickers is a Colombian e-commerce that needs to accept payments via Wompi, a Colombian payment gateway. Wompi offers a widget-based checkout that handles payment capture client-side, requiring a hybrid backend/frontend integration approach.

**Wompi Widget Approach:** The widget checkout renders a payment form on the storefront that captures payments immediately. The backend provider handles session creation, signature generation, webhook processing, and refunds.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Payment Flow                              │
├─────────────────────────────────────────────────────────────┤
│  1. Cart → Backend creates payment session                   │
│     (initiatePayment generates reference + signature)        │
│                                                               │
│  2. Frontend receives session data                            │
│     → Opens Wompi Widget with amount, reference, signature   │
│                                                               │
│  3. Customer pays in Wompi Widget                             │
│     → Wompi captures payment immediately                     │
│                                                               │
│  4. Wompi sends webhook                                       │
│     → Backend validates webhook (getWebhookActionAndData)    │
│     → Payment status updated in Medusa                        │
│                                                               │
│  5. Order completed                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## File Structure

```
medusa-wompi/
├── src/
│   └── providers/
│       └── wompi/
│           ├── index.ts              # Module provider definition
│           ├── service.ts            # Main payment provider service
│           ├── types.ts              # TypeScript interfaces
│           └── utils/
│               ├── signature.ts      # Integrity signature (SHA256)
│               └── client.ts         # Wompi API client wrapper
```

---

## Implementation Details

### 1. Provider Definition (`src/providers/wompi/index.ts`)

```typescript
import WompiPaymentProviderService from "./service"
import { ModuleProvider, Modules } from "@medusajs/framework/utils"

export default ModuleProvider(Modules.PAYMENT, {
  services: [WompiPaymentProviderService],
})
```

### 2. Types (`src/providers/wompi/types.ts`)

```typescript
export type WompiOptions = {
  public_key: string        // For frontend widget
  private_key: string       // For API calls (refunds, voids)
  integrity_key: string     // For signature generation
  environment: "sandbox" | "production"
  webhook_secret?: string   // For webhook validation (optional)
}

export type WompiTransactionStatus =
  | "APPROVED"
  | "DECLINED"
  | "VOIDED"
  | "PENDING"
  | "ERROR"

export type WompiWebhookEvent = {
  event: string
  data: {
    transaction: {
      id: string
      status: WompiTransactionStatus
      amount_in_cents: number
      reference: string
      currency: string
      payment_method_type: string
      created_at: string
    }
  }
  timestamp: number
  signature?: string
}
```

### 3. Signature Utility (`src/providers/wompi/utils/signature.ts`)

Wompi requires an integrity signature for security. The signature is a SHA256 hash of:
`reference + amount_in_cents + currency + integrity_key`

```typescript
import crypto from "crypto"

export function generateIntegritySignature(
  reference: string,
  amountInCents: number,
  currency: string,
  integrityKey: string
): string {
  const payload = `${reference}${amountInCents}${currency}${integrityKey}`
  return crypto.createHash("sha256").update(payload).digest("hex")
}
```

### 4. Wompi API Client (`src/providers/wompi/utils/client.ts`)

```typescript
import { Logger } from "@medusajs/framework/types"
import { WompiOptions, WompiTransactionStatus } from "../types"

export class WompiClient {
  private baseUrl: string
  private privateKey: string
  private logger: Logger

  constructor(options: WompiOptions, logger: Logger) {
    this.baseUrl = options.environment === "sandbox"
      ? "https://sandbox.wompi.co/v1"
      : "https://production.wompi.co/v1"
    this.privateKey = options.private_key
    this.logger = logger
  }

  // Get transaction by ID
  async getTransaction(transactionId: string): Promise<any> {
    // GET /transactions/{id}
  }

  // Refund transaction
  async refund(transactionId: string, amountInCents?: number): Promise<any> {
    // POST /transactions/{id}/refund
  }

  // Void transaction (if supported)
  async void(transactionId: string): Promise<any> {
    // POST /transactions/{id}/void
  }
}
```

### 5. Main Service (`src/providers/wompi/service.ts`)

**Key Implementation Points:**

1. **Logger Injection:** Use constructor injection pattern
   ```typescript
   type InjectedDependencies = { logger: Logger }
   constructor(container: InjectedDependencies, options: WompiOptions)
   ```

2. **Static Identifier:** Required for provider registration
   ```typescript
   static identifier = "wompi"
   ```

3. **validateOptions:** Validate required configuration on startup

---

## Required Methods Implementation

### initiatePayment

- Generate unique payment reference (use payment session ID)
- Calculate amount in cents (Wompi requires cents: amount * 100)
- Generate integrity signature
- Return session data including:
  - reference (session ID)
  - amount_in_cents
  - currency (COP)
  - signature.integrity
  - public_key (for frontend widget)

```typescript
async initiatePayment(input: InitiatePaymentInput): Promise<InitiatePaymentOutput> {
  const reference = input.context.id // payment session ID
  const amountInCents = Math.round(Number(input.amount) * 100)
  const signature = generateIntegritySignature(
    reference,
    amountInCents,
    "COP",
    this.options_.integrity_key
  )

  this.logger_.info(`[Wompi] Initiating payment: reference=${reference}, amount=${amountInCents}`)

  return {
    data: {
      reference,
      amount_in_cents: amountInCents,
      currency: "COP",
      public_key: this.options_.public_key,
      signature: { integrity: signature },
      environment: this.options_.environment,
    }
  }
}
```

### updatePayment

- Re-generate signature with new amount
- Return updated session data

### deletePayment

- Wompi doesn't support pre-payment cancellation
- Return success without action

### authorizePayment

- Wompi captures immediately on widget completion
- Return status as "authorized" (payment will be captured via webhook)
- Store transaction ID from widget response

```typescript
async authorizePayment(input: AuthorizePaymentInput): Promise<AuthorizePaymentOutput> {
  // Wompi captures payment immediately in widget
  // This method is called after webhook confirms payment
  this.logger_.info(`[Wompi] Authorizing payment: ${input.data?.reference}`)

  return {
    status: "authorized",
    data: {
      ...input.data,
      wompi_transaction_id: input.data?.transaction_id,
    }
  }
}
```

### capturePayment

- Wompi already captures in widget
- Return existing data (no additional API call needed)

```typescript
async capturePayment(input: CapturePaymentInput): Promise<CapturePaymentOutput> {
  this.logger_.info(`[Wompi] Capturing payment: ${input.data?.wompi_transaction_id}`)

  // Payment already captured by Wompi widget
  return {
    data: input.data
  }
}
```

### refundPayment

- Call Wompi refund API with transaction ID
- Support partial refunds

```typescript
async refundPayment(input: RefundPaymentInput): Promise<RefundPaymentOutput> {
  const transactionId = input.data?.wompi_transaction_id
  const refundAmount = Math.round(Number(input.amount) * 100)

  this.logger_.info(`[Wompi] Refunding payment: ${transactionId}, amount=${refundAmount}`)

  const result = await this.client_.refund(transactionId, refundAmount)

  return {
    data: {
      ...input.data,
      refund_id: result.id,
    }
  }
}
```

### cancelPayment

- Call Wompi void API if transaction not yet settled

```typescript
async cancelPayment(input: CancelPaymentInput): Promise<CancelPaymentOutput> {
  const transactionId = input.data?.wompi_transaction_id

  this.logger_.info(`[Wompi] Canceling payment: ${transactionId}`)

  await this.client_.void(transactionId)

  return { data: input.data }
}
```

### retrievePayment

- Query Wompi transaction API

```typescript
async retrievePayment(input: RetrievePaymentInput): Promise<RetrievePaymentOutput> {
  const transactionId = input.data?.wompi_transaction_id

  const transaction = await this.client_.getTransaction(transactionId)

  return {
    data: {
      ...input.data,
      status: transaction.status,
    }
  }
}
```

### getWebhookActionAndData

- Parse Wompi webhook event
- Map Wompi status to Medusa action
- Extract session ID from transaction reference

```typescript
async getWebhookActionAndData(
  payload: ProviderWebhookPayload["payload"]
): Promise<WebhookActionResult> {
  const { data } = payload
  const webhookData = data as WompiWebhookEvent

  this.logger_.info(`[Wompi] Webhook received: event=${webhookData.event}`)

  const transaction = webhookData.data.transaction
  const sessionId = transaction.reference

  try {
    switch (transaction.status) {
      case "APPROVED":
        return {
          action: PaymentActions.SUCCESSFUL,
          data: {
            session_id: sessionId,
            amount: new BigNumber(transaction.amount_in_cents / 100),
          }
        }
      case "DECLINED":
        return {
          action: PaymentActions.FAILED,
          data: {
            session_id: sessionId,
            amount: new BigNumber(transaction.amount_in_cents / 100),
          }
        }
      case "VOIDED":
        return {
          action: PaymentActions.CANCELED,
          data: {
            session_id: sessionId,
            amount: new BigNumber(transaction.amount_in_cents / 100),
          }
        }
      default:
        return {
          action: PaymentActions.NOT_SUPPORTED,
          data: {
            session_id: sessionId,
            amount: new BigNumber(0),
          }
        }
    }
  } catch (error) {
    this.logger_.error(`[Wompi] Webhook processing error: ${error}`)
    return {
      action: PaymentActions.FAILED,
      data: {
        session_id: sessionId || "",
        amount: new BigNumber(0),
      }
    }
  }
}
```

---

## Plugin Configuration

Payment provider options are configured statically via `medusa-config.ts` using environment variables (standard Medusa approach). The admin dashboard enables/disabling providers per region.

```typescript
// medusa-config.ts
module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/payment",
      options: {
        providers: [
          {
            resolve: "@yeligoth/medusa-wompi/providers/wompi",
            id: "wompi",
            options: {
              public_key: process.env.WOMPI_PUBLIC_KEY,
              private_key: process.env.WOMPI_PRIVATE_KEY,
              integrity_key: process.env.WOMPI_INTEGRITY_KEY,
              environment: process.env.WOMPI_ENVIRONMENT || "sandbox",
            }
          }
        ]
      }
    }
  ]
})
```

---

## Critical Differences from Other Providers

| Aspect           | Wompi                             | Stripe/PayPal               |
|------------------|-----------------------------------|-----------------------------|
| Amount format    | Cents (amount * 100)              | Medusa stores as-is         |
| Capture timing   | Immediate (widget handles)        | Separate authorize → capture|
| Currency         | COP only                          | Multi-currency              |
| Signature        | Required integrity hash           | Optional                    |
| Payment methods  | Cards, PSE, Nequi                 | Global cards                |

---

## Verification Steps

1. **Build the plugin:**
   ```bash
   cd medusa-wompi && yarn build
   ```

2. **Publish locally:**
   ```bash
   yarn medusa plugin:publish
   ```

3. **Install in test Medusa app:**
   ```bash
   yarn medusa plugin:add @yeligoth/medusa-wompi
   ```

4. **Configure environment variables:**
   - `WOMPI_PUBLIC_KEY` (sandbox: `pub_test_X0zDA9xoKdePzhd8a0x9HAez7HgGO2fH`)
   - `WOMPI_PRIVATE_KEY`
   - `WOMPI_INTEGRITY_KEY`
   - `WOMPI_ENVIRONMENT=sandbox`

5. **Test webhook endpoint:**
   - URL: `http://localhost:9000/hooks/payment/wompi/wompi`
   - Send mock webhook with `transaction.status=APPROVED`

6. **Check logs:**
   - Logger will output `[Wompi]` prefixed messages
   - Check for initialization, session creation, and webhook processing

---

## Environment Variables Required

```bash
# .env in Medusa application
WOMPI_PUBLIC_KEY=pub_test_xxx
WOMPI_PRIVATE_KEY=priv_test_xxx
WOMPI_INTEGRITY_KEY=integrity_test_xxx
WOMPI_ENVIRONMENT=sandbox
```

---

## References

- [Medusa Payment Provider Reference](https://docs.medusajs.com/resources/references/payment/provider)
- [Wompi Widget Checkout Documentation](https://docs.wompi.co/docs/colombia/widget-checkout-web/)
- [Medusa Container Documentation](https://docs.medusajs.com/learn/fundamentals/modules/container)
- [Medusa Payment Webhook Events](https://docs.medusajs.com/resources/commerce-modules/payment/webhook-events)

---

## Implementation Progress

### Backend Plugin Files (medusa-wompi)

| File | Status | Notes |
|------|--------|-------|
| `src/providers/wompi/index.ts` | ✅ Done | Module provider definition |
| `src/providers/wompi/service.ts` | ✅ Done | Main service with all payment methods (14KB) |
| `src/providers/wompi/types.ts` | ✅ Done | TypeScript interfaces |
| `src/providers/wompi/utils/signature.ts` | ✅ Done | SHA256 integrity signature |
| `src/providers/wompi/utils/client.ts` | ✅ Done | Wompi API client wrapper |
| `src/admin/` | ✅ Done | Admin dashboard customizations |
| `src/api/` | ✅ Done | API routes |
| Plugin package.json | ✅ Done | Published as `@yeligoth/medusa-wompi` |

### Backend Configuration (caribestickers-store)

| File | Status | Notes |
|------|--------|-------|
| `medusa-config.ts` | ✅ Done | Wompi provider configured |
| `.env` | ⏳ Pending | Environment variables need verification |
| Plugin installation | ✅ Done | Installed via yalc in `.yalc/@yeligoth/medusa-wompi` |

### Frontend Integration (caribestickers-store-storefront)

| File | Status | Notes |
|------|--------|-------|
| `src/lib/constants.tsx` | ✅ Done | Wompi constants defined |
| `src/modules/checkout/components/payment-container/wompi-container.tsx` | ✅ Done | Wompi payment container |
| `src/modules/checkout/components/payment-wrapper/wompi-wrapper.tsx` | ✅ Done | Wompi wrapper component |
| `src/modules/checkout/components/payment/index.tsx` | ✅ Done | Payment component updated |
| `src/modules/checkout/components/payment-wrapper/index.tsx` | ✅ Done | Payment wrapper updated |
| `src/modules/checkout/components/payment-button/index.tsx` | ✅ Done | Payment button updated |

### Remaining Tasks

| Task | Status | Priority |
|------|--------|----------|
| End-to-end testing | ⏳ Pending | High |
| Webhook endpoint testing with real Wompi | ⏳ Pending | High |
| Verify environment variables are set correctly | ⏳ Pending | High |
| Production deployment | ⏳ Pending | Medium |

---

## Notes

- Plugin installed via yalc in `caribestickers-store/.yalc/@yeligoth/medusa-wompi`
- Also linked in `node_modules/@yeligoth/medusa-wompi`
- COP currency only supported currently
- Frontend Wompi Widget integration completed