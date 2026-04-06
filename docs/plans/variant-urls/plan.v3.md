---
model: kimi-k2.5
code_agent: Claude Code
created_at: 2026-04-05
updated_at: 2026-04-05
merged_from: [plan.v1.md, plan.v2.md]
---

# Implementation Plan: Unique Variant URLs (v3)

## Overview

Enable shareable product variant URLs by adding a `?variant={variantId}` query parameter when a complete variant is selected. Users can share direct links to specific product variants.

## Context

This feature allows customers to share direct links to specific product variants (e.g., a specific size and color combination). When users open a product page with the `variant` query parameter, the corresponding variant's options should be preselected automatically.

## Current Implementation

The product variant selection is handled in:
- **File:** `caribestickers-store-storefront/src/modules/products/components/product-actions/index.tsx`

**Current behavior:**
- Component tracks selected options via `useState`
- `selectedVariant` is computed via `useMemo` by matching selected options against variant options
- When product has exactly 1 variant, options are preselected on mount
- Options are selected via `OptionSelect` component buttons

## File to Modify

`caribestickers-store-storefront/src/modules/products/components/product-actions/index.tsx`

## Implementation Steps

### 1. Add imports

Add to existing imports:
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

Replace the existing `useEffect` (lines 41-46) with:
```typescript
useEffect(() => {
  const variantIdFromUrl = searchParams.get("variant")

  if (variantIdFromUrl) {
    // Look up variant by ID from URL
    const variantFromUrl = product.variants?.find(
      (v) => v.id === variantIdFromUrl
    )
    if (variantFromUrl) {
      const variantOptions = optionsAsKeymap(variantFromUrl.options)
      setOptions(variantOptions ?? {})
    }
  } else if (product.variants?.length === 1) {
    // Existing behavior: preselect single variant
    const variantOptions = optionsAsKeymap(product.variants[0].options)
    setOptions(variantOptions ?? {})
  }
}, [product.variants, searchParams])
```

### 4. Add effect to update URL when variant is found

Add new effect after the mount effect:
```typescript
useEffect(() => {
  if (selectedVariant?.id) {
    const params = new URLSearchParams(searchParams.toString())
    params.set("variant", selectedVariant.id)
    router.replace(pathname + "?" + params.toString(), { scroll: false })
  }
}, [selectedVariant, router, pathname, searchParams])
```

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
    params = new URLSearchParams(searchParams.toString())
    params.set('variant', selectedVariant.id)
    router.replace(pathname + '?' + params.toString(), { scroll: false })
```

## Behavior Matrix

| Scenario | Expected Behavior |
|----------|------------------|
| User opens `/products/sticker-1?variant=variant_123` | Preselect options matching variant_123 |
| User opens `/products/sticker-1` (no param, 1 variant) | Preselect the single variant (existing behavior) |
| User opens `/products/sticker-1` (no param, multiple variants) | No preselection, user must choose |
| User clicks options until valid variant found | Update URL to `?variant={id}` via `router.replace()` |
| Invalid variant ID in URL | Ignore, allow normal selection |
| User changes options to different valid variant | URL updates to new variant ID |

## Design Decisions

1. **URL update timing**: Update URL only when a complete variant is found (cleaner URLs, no partial states)
2. **Navigation method**: Use `router.replace()` instead of `router.push()` to avoid polluting browser history
3. **Scroll behavior**: Pass `{ scroll: false }` to prevent unwanted scroll jumps when URL updates
4. **Query param preservation**: Use `searchParams.toString()` to preserve any existing query parameters
5. **Timing**: Only update URL when a complete valid variant is found (`selectedVariant?.id` exists)
6. **Precedence**: URL variant parameter takes precedence over single-variant auto-selection

## Verification

Test the following scenarios:
1. Open product page without variant param → select options → verify URL updates with `?variant={id}`
2. Copy URL with variant param → open in new tab → verify correct variant is preselected
3. Open product with single variant (no URL param) → verify variant is preselected
4. Enter invalid variant ID in URL → verify normal selection works
5. Navigate back/forward in browser → verify variant selection syncs with URL

---

## Merge Notes: Differences Between Plans

### What was improved from v1 → v3:

| Aspect | v1 (opencode) | v3 (merged) |
|--------|---------------|-------------|
| **Structure** | Simple, direct | Organized with clear sections |
| **Scroll behavior** | Not mentioned | Explicit `{ scroll: false }` added |
| **Behavior matrix** | 4 scenarios | 6 scenarios (added multi-variant and variant change cases) |
| **Current implementation** | Brief | Detailed explanation of existing behavior |
| **Verification** | Not included | Added test scenarios section |
| **Design decisions** | 3 points | 6 points with detailed rationale |
| **File paths** | Relative (`src/modules/...`) | Absolute (`caribestickers-store-storefront/src/modules/...`) |

### Key improvements made:

1. **Added `scroll: false`**: Critical UX improvement from v2 to prevent page jumping when URL updates
2. **Expanded behavior matrix**: Added "multiple variants" and "user changes options" scenarios for complete coverage
3. **Added verification section**: Practical test steps from v2 to ensure implementation quality
4. **Kept v1's clarity**: Maintained the concise structure and logic flow diagram from v1
5. **Added metadata**: Included model, agent, dates, and merge source for traceability

### What was preserved from v1:

- Concise "Implementation Steps" format
- Clear "Logic Flow" diagram
- Focus on the single file to modify
- Direct code snippets without excessive explanation

### What was incorporated from v2:

- Complete file paths for clarity
- `scroll: false` parameter
- Extended behavior scenarios
- Design decisions with rationale
- Verification test cases
- Current implementation context
