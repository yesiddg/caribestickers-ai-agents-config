# CLAUDE.md

This guide provides essential information for agentic coding agents working with this Medusa v2 e-commerce monorepo containing both a headless backend and Next.js storefront.
The name of the bussiness is caribestickers. It's an ecommerce that sells sticker packs.

## Project Structure

This project has two separate applications:

- **`caribestickers-store/`** — Medusa v2 headless commerce backend (port 9000)
- **`caribestickers-store-storefront/`** — Next.js 15 storefront frontend (port 8000)

Each directory has its own `package.json`, `node_modules`, and must be run independently. There is no workspace/shared tooling at the root level.

## Strict rules

- Don't restart the servers at all.
- Don't make commits.
- Don't push to the remote repository.
- Don't open pull requests.
- Don't merge or rebase branches.

## Build/Lint/Test Commands

### Backend (`caribestickers-store`)

A detailed `CLAUDE.md` lives inside this directory. Key commands:

```bash
# Development
yarn dev            # Start development server with hot reload
yarn build          # Build for production

# Database
npx medusa db:migrate              # Run pending migrations
npx medusa db:generate <module>    # Generate migration for a module
yarn seed           # Seed DB with demo products, regions, shipping

# Testing
yarn test:unit                     # Run unit tests
yarn test:integration:http         # Run HTTP integration tests
yarn test:integration:modules      # Run module integration tests

# Running a single test
yarn test:unit src/modules/products/__tests__/product.unit.spec.ts
yarn test:integration:http integration-tests/http/products.spec.ts
yarn test:integration:modules src/modules/products/__tests__/products.spec.ts
```

**Environment**: Copy `.env.template` → `.env`. Requires `DATABASE_URL`, `JWT_SECRET`, `COOKIE_SECRET`, CORS settings.

#### Backend Architecture Summary

Medusa v2 uses isolated modules, file-based API routes, composable workflows, and event subscribers. See `caribestickers-store/CLAUDE.md` for the full breakdown of patterns, dependency injection, module creation workflow, and testing setup.

### Frontend (`caribestickers-store-storefront`)

```bash
# Development
yarn dev     # Next.js dev server with Turbopack on port 8000
yarn build   # Production build
yarn start   # Serve production build on port 8000

# Linting
yarn lint    # Run ESLint

# Note: No dedicated test suite in frontend currently
```

**Environment**: `.env.local` requires `NEXT_PUBLIC_MEDUSA_BACKEND_URL` (backend URL) and `NEXT_PUBLIC_STRIPE_KEY`.

#### Frontend Architecture

**Framework**: Next.js 15 App Router with React 19 RC, TypeScript, Tailwind CSS v3, Shadcn/ui.

**Routing**: All store routes are nested under `[countryCode]` for multi-region support (e.g., `/us/products/sticker-1`). Country resolution happens in `src/middleware.ts`.

**Directory layout:**
- `src/app/[countryCode]/(main)/` — Store pages (home, products, collections, categories, cart, account, orders)
- `src/app/[countryCode]/(checkout)/` — Isolated checkout layout/pages
- `src/modules/` — Feature modules (account, cart, checkout, products, layout, etc.), each containing components and logic for that domain
- `src/lib/data/` — Server-side data fetching functions using `@medusajs/js-sdk`
- `src/lib/context/` — React contexts for client-side state
- `src/lib/util/` — Utility functions (price formatting, locale, currency, country)
- `src/app/ui/` — 40+ Shadcn/ui base components

**Path aliases** (`tsconfig.json`):
- `@lib/*` → `src/lib/*`
- `@modules/*` → `src/modules/*`
- `@pages/*` → `src/app/*`

#### Code Style Guidelines

##### Imports

1. Use absolute imports with path aliases when possible:
   - `@lib/*` → `src/lib/*`
   - `@modules/*` → `src/modules/*`
   - `@pages/*` → `src/app/*`

2. Import order preference:
   - React/external libraries
   - Medusa packages
   - Internal modules/libraries
   - Relative imports
   - Type imports last

3. Import destructuring:
   - Single export: `import Thing from "module"`
   - Multiple exports: `import { A, B, C } from "module"`
   - Namespace imports: `import * as utils from "module"` when importing many

##### Formatting

1. TypeScript/JavaScript follows standard linting from:
   - Backend: Strict TypeScript with SWC
   - Frontend: Next.js core-web-vitals ESLint config

2. General formatting rules:
   - 2 space indentation
   - Semicolon usage consistent with existing code
   - Line length maximum 100 characters
   - Trailing commas in multiline objects/arrays

##### Types

1. Use TypeScript for all new code
2. Define interfaces for complex objects
3. Use proper typing for function parameters and return values
4. Leverage Medusa's built-in types when available
5. Strict mode enabled in both projects

##### Naming Conventions

1. Variables/functions: camelCase
2. Classes/Components: PascalCase
3. Constants: UPPER_CASE with underscores
4. Files: kebab-case for routing files, otherwise descriptive names
5. Module names: plural nouns when representing collections (products, users)

**Data fetching pattern**: Server components call functions from `src/lib/data/` which use `@medusajs/js-sdk` to query the backend. Client components use React context or hook into server action results.

## Error Handling

1. Backend:
   - Use Medusa's error classes from `@medusajs/utils`
   - Handle errors with try/catch blocks at service/workflow boundaries
   - Throw appropriate HTTP status codes via API routes

2. Frontend:
   - Use React error boundaries for UI components
   - Handle API errors gracefully with user feedback
   - Implement retry mechanisms where appropriate

## Documentation

1. Comment complex logic with explanatory notes
2. Export JSDoc/TypeDoc for public APIs
3. Follow existing patterns in the codebase

## Key Integration Points

- Frontend → Backend: REST API over HTTP, configured via `NEXT_PUBLIC_MEDUSA_BACKEND_URL`
- Multi-region: 7 European countries (GB, DE, DK, SE, FR, ES, IT), currencies EUR and USD
- Images: Served from S3 (configured in `next.config.js` `remotePatterns`)
- Authentication: Medusa handles auth; frontend stores session via cookies managed by the SDK
