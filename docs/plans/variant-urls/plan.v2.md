---
model: kimi-k2.5
code_agent: Claude Code
created_at: 2026-04-05
updated_at: 2026-04-05
---

# Plan: Unique Variant URLs for Product Pages

## Context

Enable shareable product variant URLs by adding a `?variant={variantId}` query parameter when a user selects options that result in a valid variant. When users open a product page with this query parameter, the corresponding variant's options should be preselected.

This allows customers to share direct links to specific product variants (e.g., a specific size and color combination).

## Current Implementation

The product variant selection is handled in:
- **File:** `caribestickers-store-storefront/src/modules/products/components/product-actions/index.tsx`

**Current behavior:**
- Component tracks selected options via `useState`
- `selectedVariant` is computed via `useMemo` by matching selected options against variant options
- When product has exactly 1 variant, options are preselected on mount
- Options are selected via `OptionSelect` component buttons

## Changes Required

### File: `caribestickers-store-storefront/src/modules/products/components/product-actions/index.tsx`

#### 1. Add Next.js navigation hooks import

Add to existing imports:
```typescript
import { useParams, useSearchParams, useRouter, usePathname } from "next/navigation"
```

#### 2. Initialize URL-related hooks

Add after existing hooks:
```typescript
const searchParams = useSearchParams()
const router = useRouter()
const pathname = usePathname()
```

#### 3. Add effect to read variant from URL on mount

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

#### 4. Add effect to update URL when variant is selected

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

1. **URL update method:** Use `router.replace()` instead of `router.push()` to avoid polluting browser history with every option change
2. **Scroll behavior:** Pass `{ scroll: false }` to prevent unwanted scroll jumps when URL updates
3. **Query param preservation:** Use `searchParams.toString()` to preserve any existing query parameters
4. **Timing:** Only update URL when a complete valid variant is found (`selectedVariant?.id` exists)
5. **Precedence:** URL variant parameter takes precedence over single-variant auto-selection

## Verification

Test the following scenarios:
1. Open product page without variant param → select options → verify URL updates with `?variant={id}`
2. Copy URL with variant param → open in new tab → verify correct variant is preselected
3. Open product with single variant (no URL param) → verify variant is preselected
4. Enter invalid variant ID in URL → verify normal selection works
5. Navigate back/forward in browser → verify variant selection syncs with URL

## Critical Files

- `caribestickers-store-storefront/src/modules/products/components/product-actions/index.tsx` - Main implementation file
