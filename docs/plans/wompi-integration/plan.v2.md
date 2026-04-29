---
model: glm-5
code_agent: Claude Code
created_at: 2026-04-28
updated_at: 2026-04-28
based_on: plan.v1.md
implementation_status: completed
---

# Implementation Plan: Wompi Payment Provider Integration (v2)

## Overview

This plan documents the completed Wompi payment provider integration for Medusa v2, including the Shopify-like checkout flow that removes the Review step for automatic order creation after payment approval.

## Context

**Business:** Caribestickers - Colombian e-commerce selling sticker packs
**Payment Gateway:** Wompi (Colombian payment processor - Cards, PSE, Nequi)
**Goal:** Accept payments via Wompi with seamless checkout experience

---

## Architecture

### Hybrid Backend/Frontend Approach

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
│  4. Payment approved → placeOrder() called automatically     │
│     → Redirect to order confirmation                         │
│                                                               │
│  5. Wompi sends webhook → Backend validates                  │
│     → Payment status updated in Medusa                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Backend Plugin (medusa-wompi)

### File Structure

```
medusa-wompi/
├── src/
│   ├── providers/wompi/
│   │   ├── index.ts              # Module provider definition
│   │   ├── service.ts            # Main payment provider service
│   │   ├── types.ts              # TypeScript interfaces
│   │   └── utils/
│   │       ├── signature.ts      # SHA256 integrity signature
│   │       └── client.ts         # Wompi API client wrapper
│   ├── admin/                    # Admin dashboard customizations
│   ├── api/                      # API routes
│   └── workflows/                # Medusa workflows
├── package.json                  # Published as @yeligoth/medusa-wompi
└── README.md
```

### Key Backend Components

| Component | Purpose |
|-----------|---------|
| `service.ts` | Payment provider service with `initiatePayment`, `getWebhookActionAndData`, `refundPayment` |
| `signature.ts` | SHA256 integrity signature: `reference + amount_in_cents + currency + integrity_key` |
| `client.ts` | Wompi API wrapper for refunds, voids, transaction queries |
| `types.ts` | TypeScript interfaces for WompiOptions, WompiWebhookEvent |

### Configuration

```typescript
// medusa-config.ts
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
```

---

## Frontend Integration (caribestickers-store-storefront)

### Shopify-like Checkout Flow

**Key Change:** Removed Review step - order created automatically when Wompi payment approved.

### Modified Files

| File | Change |
|------|--------|
| `wompi-container.tsx` | Added `placeOrder()` call in widget callback, removed `isPaid` state |
| `payment/index.tsx` | Removed "Continue to review" button, added inline PaymentButton for Stripe/Manual |
| `checkout-form/index.tsx` | Removed `<Review cart={cart} />` component |
| `payment-button/index.tsx` | Removed `WompiPaymentButton` export |
| `review/index.tsx` | **Deleted** |

### WompiContainer Implementation

```tsx
// src/modules/checkout/components/payment-container/wompi-container.tsx
import { placeOrder } from "@lib/data/cart"

const WompiContainer = ({ ... }) => {
  const [submitting, setSubmitting] = useState(false)
  
  const handleOpenWidget = () => {
    const widget = new window.WidgetCheckout(config)
    
    widget.open(async (response) => {
      if (response.transaction?.status === "APPROVED") {
        setSubmitting(true)
        try {
          await placeOrder()  // Auto-create order → redirect to confirmation
        } catch (err) {
          setError(`Payment approved but order creation failed. Contact support with transaction ID: ${response.transaction.id}`)
          setSubmitting(false)
        }
      } else if (response.transaction) {
        setError(`Payment ${response.transaction.status.toLowerCase()}. Please try again.`)
      }
    })
  }
  
  return (
    <button onClick={handleOpenWidget} disabled={submitting}>
      {submitting ? "Processing payment..." : "Pay with Wompi"}
    </button>
  )
}
```

### Payment Flow Comparison

| Provider | Flow |
|----------|------|
| **Wompi** | Select → Widget → Payment → Auto order → Confirmation |
| **Stripe** | Select → Enter card → Place order button → Payment → Confirmation |
| **Manual** | Select → Place order button → Confirmation |

---

## Environment Variables

```bash
# Backend (.env)
WOMPI_PUBLIC_KEY=pub_test_xxx
WOMPI_PRIVATE_KEY=priv_test_xxx
WOMPI_INTEGRITY_KEY=integrity_test_xxx
WOMPI_ENVIRONMENT=sandbox
```

---

## Verification Checklist

### Backend Plugin

- [x] `initiatePayment` generates reference + signature
- [x] `getWebhookActionAndData` maps Wompi status to Medusa actions
- [x] `refundPayment` supports partial refunds
- [x] Plugin installed via yalc in caribestickers-store

### Frontend Integration

- [x] Wompi widget loads with correct payment data
- [x] Payment approved → Order auto-created
- [x] Redirect to confirmation page
- [x] Error handling for payment/order failures
- [x] Review step removed from checkout

### End-to-End Testing

- [ ] Test complete checkout flow with sandbox
- [ ] Test webhook endpoint with real Wompi events
- [ ] Test refund flow via admin dashboard
- [ ] Test with production credentials

---

## Key Differences from Other Providers

| Aspect | Wompi | Stripe |
|--------|-------|--------|
| Amount format | Cents (amount × 100) | Medusa stores as-is |
| Capture timing | Immediate (widget handles) | Authorize → Capture |
| Currency | COP only | Multi-currency |
| Signature | Required SHA256 integrity hash | Optional |
| Order creation | Automatic on payment approval | Manual button click |

---

## Edge Cases Handled

| Scenario | Handling |
|----------|----------|
| Payment approved but order creation fails | Error message with transaction ID for support |
| User closes widget without paying | Stay on payment step, allow retry |
| Payment declined | Error message, allow retry |
| Missing shipping address/method | Prerequisite validation |

---

## References

- [Medusa Payment Provider](https://docs.medusajs.com/resources/references/payment/provider)
- [Wompi Widget Checkout](https://docs.wompi.co/docs/colombia/widget-checkout-web/)
- [Medusa Container](https://docs.medusajs.com/learn/fundamentals/modules/container)

---

## Installation Notes

Plugin installed locally via yalc:
```bash
cd medusa-wompi && yarn build
yalc publish
cd ../caribestickers-store && yalc add @yeligoth/medusa-wompi
```

Located in: `caribestickers-store/.yalc/@yeligoth/medusa-wompi`