---
model: glm-5
code_agent: opencode
created_at: 2026-04-05
updated_at: 2026-04-05
---

# Implementation Plan: Unique Variant URLs

## Overview

Enable shareable product variant URLs by adding a `?variant={variantId}` query parameter when a complete variant is selected. Users can share direct links to specific product variants.

## File to Modify

`src/modules/products/components/product-actions/index.tsx`

## Implementation Steps

### 1. Add imports

```tsx
import { useParams, useSearchParams, useRouter, usePathname } from "next/navigation"
```

### 2. Initialize hooks

Add after existing `useParams`:

```tsx
const searchParams = useSearchParams()
const router = useRouter()
const pathname = usePathname()
```

### 3. Add effect to read variant from URL on mount

- Runs once on mount
- Checks for `variant` query param
- If found, looks up variant by ID and sets its options
- Falls back to existing single-variant preselection logic

### 4. Add effect to update URL when variant is found

- Watches `selectedVariant`
- When a valid variant is selected, updates URL with `router.replace()`
- Preserves other query params

### 5. Add helper function

Function to build query string while preserving existing params.

## Logic Flow

```
On Mount:
  variantId = searchParams.get('variant')
  if variantId exists:
    variant = product.variants.find(v => v.id === variantId)
    if variant exists:
      setOptions(optionsAsKeymap(variant.options))
  else if product.variants.length === 1:
    // existing behavior
    setOptions(optionsAsKeymap(product.variants[0].options))

On selectedVariant change:
  if selectedVariant exists:
    params = new URLSearchParams(searchParams)
    params.set('variant', selectedVariant.id)
    router.replace(pathname + '?' + params.toString())
```

## Behavior Matrix

| Scenario | Behavior |
|----------|----------|
| User opens `/products/sticker-1?variant=variant_123` | Preselect options for variant_123 |
| User opens `/products/sticker-1` (no param) | Preselect first variant if only 1 variant exists |
| User clicks options until variant found | Update URL with `?variant={id}` |
| Invalid variant ID in URL | Ignore, fall back to default behavior |

## Design Decisions

- **URL update timing**: Update URL only when a complete variant is found (cleaner URLs, no partial states)
- **Navigation method**: Use `router.replace()` to avoid polluting browser history
- **Query param preservation**: Preserve any existing query params when adding `variant`