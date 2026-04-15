# Copilot Workspace Instructions

> **These rules are always active for every Copilot conversation in this workspace.**
> They govern architecture, conventions, and how features should be built.
> For one-time project initialization, use `SETUP.md` instead — do not re-run bootstrap or setup steps unless explicitly instructed.

---

## Project Overview

An e-commerce website built with Next.js 16 (App Router). It has a public storefront visible to everyone, authentication pages (login/signup), and a protected account area for order history and settings. The project uses a **design token system** with Tailwind v4 integration, Storybook for component documentation, and an MCP server that helps AI assistants work with the design system.

## Stack

> The versions below were current when this document was written. `bun create next-app@latest` may install newer versions. Always verify actual installed versions via `cat node_modules/<pkg>/package.json | grep '"version"'` and use `node_modules/next/dist/docs/` as the authoritative reference for the installed version.

- **Next.js 16+** (App Router only — no Pages Router)
- **React 19.2.4**
- **Tailwind CSS v4** — no `tailwind.config.js`; all config lives in CSS via `@theme`
- **TypeScript 5** (strict mode, `moduleResolution: bundler`)
- **Bun** — use `bun` for all package management and scripts (not npm/yarn)
- **ESLint 9** flat config (`eslint.config.mjs`)
- **Storybook 8** (`@storybook/nextjs` framework)
- **MCP SDK** (`@modelcontextprotocol/sdk`)
- **Redux Toolkit (RTK)** + `react-redux` — client-side global mutable state only
- **axios** — HTTP client for external API calls

> ⚠️ Tailwind v4 is a breaking change from v3. Before writing any Tailwind config, read `node_modules/next/dist/docs/01-app/01-getting-started/11-css.md`.
> ⚠️ Next.js 16 has breaking changes. Before writing any Next.js code, read relevant guides in `node_modules/next/dist/docs/`.
> ⚠️ **Always use the bundled Next.js docs at `node_modules/next/dist/docs/` as the authoritative reference for all Next.js questions.** Never rely on training data or online docs — they describe older versions. The bundled docs reflect the exact version installed.
> ⚠️ **Never use a bare `<a>` tag for internal navigation — always use `<Link>` from `'next/link'`.** Using `<a>` bypasses client-side routing.
> ⚠️ **Never use `<img>` — always use `<Image>` from `'next/image'`.** Raw `<img>` tags skip Next.js image optimisation, lazy loading, and size hints.
> ⚠️ **Never add `<meta>` tags manually in JSX** — always use the `metadata` export or `generateMetadata` function from `layout.tsx` or `page.tsx`. Next.js generates all `<head>` tags automatically.
> ⚠️ **Never use a bare `<script>` tag** — always use `<Script>` from `'next/script'` with the correct `strategy` prop.
> ⚠️ **Do not assume — verify.** Before writing code that touches any Next.js API, routing convention, or component behaviour, read the relevant doc in `node_modules/next/dist/docs/`. If you make an assumption, state it explicitly and confirm it against the bundled docs or existing codebase before writing the code.

---

## App Routing Architecture

Routes are organised with **Next.js Route Groups** (`(folderName)`) so each section gets its own layout without polluting the URL.

| Route Group | URL prefix                                          | Layout                          | Who can access              |
| ----------- | --------------------------------------------------- | ------------------------------- | --------------------------- |
| `(public)`  | `/`, `/products`, `/products/[slug]`, `/cart`, etc. | Header (auth-aware) + Footer    | Everyone — logged in or not |
| `(auth)`    | `/login`, `/signup`                                 | Centered card, no header/footer | Unauthenticated users only  |
| `(account)` | `/account`, `/account/orders`, `/account/settings`  | Header + account nav            | Authenticated users only    |

**Key distinction from an admin panel:**

- `(public)` pages are Server Components that render for all visitors. The Header reads the session server-side and conditionally renders a guest nav (Login / Sign up) or an authenticated nav (avatar, account link).
- `(auth)` pages redirect to `/account` if the user is already logged in.
- `(account)` pages redirect to `/login` if the user is not logged in. `proxy.ts` handles both redirects.
- `StoreProvider` wraps the root `layout.tsx` — all route groups (public, auth, account) have access to Redux state. `cartSlice` and `uiSlice` are needed on public pages (Navbar cart icon, mobile menu), so scoping the store to `(account)` only would break those.

### Route file conventions (App Router)

Every route segment may include:

- `page.tsx` — the page itself (Server Component by default)
- `layout.tsx` — persistent UI wrapper for that segment and its children
- `loading.tsx` — Suspense boundary skeleton shown while the page streams
- `error.tsx` — error boundary for that segment
- `not-found.tsx` — 404 for that segment

### Route structure

```
src/app/
├── (public)/
│   ├── layout.tsx             # Public layout — Header (auth-aware) + Footer
│   ├── page.tsx               # / (home — hero, featured products)
│   ├── products/
│   │   ├── page.tsx           # /products (catalogue with filters)
│   │   └── [slug]/
│   │       └── page.tsx       # /products/[slug] (product detail)
│   ├── cart/
│   │   └── page.tsx           # /cart
│   ├── checkout/
│   │   └── page.tsx           # /checkout
│   └── about/
│       └── page.tsx           # /about
├── (auth)/
│   ├── layout.tsx             # Auth layout — centered card, no header/footer
│   ├── login/
│   │   └── page.tsx           # /login
│   └── signup/
│       └── page.tsx           # /signup
├── (account)/
│   ├── layout.tsx             # Account layout — auth-gated
│   ├── loading.tsx            # Account skeleton
│   └── account/
│       ├── page.tsx           # /account (profile)
│       ├── orders/
│       │   ├── page.tsx       # /account/orders (order history)
│       │   └── [id]/
│       │       └── page.tsx   # /account/orders/[id] (order detail)
│       └── settings/
│           └── page.tsx       # /account/settings
├── api/                       # BFF Route Handlers (server-only proxy)
│   └── [feature]/
│       └── route.ts
├── globals.css
├── tokens.css                 # GENERATED — do not edit manually
└── layout.tsx                 # Root layout — wraps <StoreProvider>, fonts, metadata
```

---

## Proxy (`proxy.ts`)

> ⚠️ **Next.js 16 renamed Middleware to Proxy.** The file is `src/proxy.ts` (inside `src/`, same level as `src/app/`). The function is named `proxy` (not `middleware`). The `middleware.ts` file convention is deprecated. Always use `proxy.ts`.

`proxy.ts` runs server-side before a request reaches any route. For this e-commerce site it handles two auth redirects:

- **`(auth)` guard** — `/login` and `/signup` redirect to `/account` if the user is already logged in (no point showing auth pages to logged-in users)
- **`(account)` guard** — all `/account/*` routes redirect to `/login` if the user is not logged in

```ts
// src/proxy.ts  — inside src/, same level as src/app/
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Read session cookie — replace 'session' with your actual cookie name
  const isLoggedIn = request.cookies.has("session");

  // (account) guard — redirect unauthenticated users to /login
  if (pathname.startsWith("/account") && !isLoggedIn) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // (auth) guard — redirect logged-in users away from /login and /signup
  if ((pathname === "/login" || pathname === "/signup") && isLoggedIn) {
    return NextResponse.redirect(new URL("/account", request.url));
  }

  return NextResponse.next();
}

export const config = {
  // Run on account and auth routes only — skip static files, api routes, _next
  matcher: ["/account/:path*", "/login", "/signup"],
};
```

**Rules:**

- `proxy.ts` must be inside `src/` (at `src/proxy.ts`) — same level as `src/app/`, never inside `app/` itself
- Export the function as `proxy` (named) or as `default` — not `middleware`
- `config.matcher` must be a static constant — no dynamic values
- Do not do slow data fetching inside `proxy.ts` — only fast cookie/header checks
- Pass data to the app via headers, cookies, or URL — not via shared globals
- Before modifying `proxy.ts`, read `node_modules/next/dist/docs/01-app/01-getting-started/16-proxy.md`

---

## SEO & Metadata

> Before writing any metadata, read `node_modules/next/dist/docs/01-app/01-getting-started/14-metadata-and-og-images.md`.

Next.js generates all `<head>` tags automatically from the `metadata` export or `generateMetadata` function. Never write `<meta>` or `<title>` tags in JSX.

### Title template — set once in root layout, inherited everywhere

```ts
// src/app/layout.tsx
export const metadata: Metadata = {
  title: {
    template: "%s | ShopName", // individual pages set the %s part
    default: "ShopName", // fallback when no page sets a title
  },
  description: "Your one-stop shop for ...",
  metadataBase: new URL(process.env.NEXT_PUBLIC_SITE_URL),
};

// src/app/(public)/products/page.tsx — static
export const metadata: Metadata = {
  title: "Products", // renders: "Products | ShopName"
};

// src/app/(public)/products/[slug]/page.tsx — dynamic
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await getProductBySlug((await params).slug);
  return {
    title: product.name, // renders: "Air Max 90 | ShopName"
    description: product.description,
    openGraph: {
      images: [{ url: product.image, width: 1200, height: 630 }],
    },
  };
}
```

### SEO helpers folder

```
src/lib/seo/
├── defaults.ts        # SITE_NAME, SITE_URL, default OG image, Twitter handle
└── helpers.ts         # buildProductMeta(), buildCategoryMeta(), buildPageMeta()
```

Helpers return typed `Metadata` objects. Pages call them instead of constructing metadata inline:

```ts
import { buildProductMeta } from "@/lib/seo/helpers";
export const generateMetadata = ({ params }) => buildProductMeta(params.slug);
```

### File-based SEO (place in `src/app/` root)

| File                  | Purpose                                                          |
| --------------------- | ---------------------------------------------------------------- |
| `favicon.ico`         | Auto-detected, no code needed                                    |
| `opengraph-image.tsx` | Default OG image for all pages (static or `ImageResponse`)       |
| `robots.ts`           | Generates `/robots.txt` at build time                            |
| `sitemap.ts`          | Generates `/sitemap.xml` — include all product and category URLs |

### Rules

- Never write `<meta>`, `<title>`, or `<link rel="canonical">` in JSX — use the metadata API
- Always set `metadataBase` in root layout — required for absolute OG image URLs
- Use `title.template` in root layout — never hardcode the site name in every page title
- `generateMetadata` runs server-side — fetch product/category data directly from `lib/services/`, using React `cache()` to avoid duplicate fetches with the page
- Static pages use `export const metadata` — dynamic pages (product detail, category) use `generateMetadata`
- OG images: static file for most pages, `opengraph-image.tsx` with `ImageResponse` for product detail
- Read `node_modules/next/dist/docs/01-app/03-api-reference/04-functions/generate-metadata.md` for the full `Metadata` field reference

---

## 3rd-Party Scripts (GTM & Trackers)

> Before adding any script, read `node_modules/next/dist/docs/01-app/02-guides/scripts.md`.

Use `<Script>` from `'next/script'` — never a bare `<script>` tag. GTM and all trackers go in root `layout.tsx` so they fire on every page.

### Strategy reference

| Strategy            | When to use                                                          |
| ------------------- | -------------------------------------------------------------------- |
| `beforeInteractive` | Consent management, critical polyfills — loads before hydration      |
| `afterInteractive`  | GTM, analytics — loads early, after some hydration (default)         |
| `lazyOnload`        | Chat widgets, non-critical trackers — loads during browser idle time |
| `worker`            | Experimental — offloads to web worker via Partytown                  |

### Analytics folder

```
src/lib/analytics/
├── GTMScript.tsx      # 'use client' — <Script> wrapper for GTM snippet
├── GTMNoScript.tsx    # <noscript> iframe fallback — placed after <body> in layout
└── index.ts           # re-exports all tracker components
```

### Usage in root layout

```tsx
// src/app/layout.tsx
import { GTMScript, GTMNoScript } from "@/lib/analytics";

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <GTMNoScript /> {/* must be first inside <body> */}
        {children}
      </body>
      <GTMScript /> {/* strategy="afterInteractive" — outside <body> is fine */}
    </html>
  );
}
```

### Rules

- `GTMScript` uses `strategy="afterInteractive"` — never `beforeInteractive` for GTM
- `GTMNoScript` renders the `<noscript><iframe>` fallback — place it as the first child of `<body>`
- GTM container ID goes in `.env` as `NEXT_PUBLIC_GTM_ID` — never hardcoded
- Add other trackers (Meta Pixel, Hotjar, etc.) as separate named components in `lib/analytics/`
- `onLoad` / `onReady` event handlers require `'use client'` on the component using them
- Read `node_modules/next/dist/docs/01-app/03-api-reference/02-components/script.md` for full `<Script>` API

The browser **never** calls external APIs directly. All external calls go through a **BFF (Backend for Frontend)** proxy using Next.js Route Handlers. Secrets (API keys, tokens) stay server-side.

### Flow

```
Client Component
  → fetch("/api/[feature]")        # calls our BFF Route Handler
    → lib/services/*.ts            # typed fetch functions
      → External API               # real backend, secrets attached server-side

Server Component (RSC)
  → lib/services/*.ts              # call external API directly (no proxy needed)
    → External API
```

### API folder structure

```
lib/
├── api/
│   ├── client.ts              # Base axios instance — sets baseURL, auth headers, interceptors
│   └── endpoints.ts           # All external API URLs as typed UPPER_CASE constants
└── services/
    ├── auth.service.ts        # login(), signup(), refreshToken()
    ├── product.service.ts     # getProducts(), getProductBySlug()
    ├── cart.service.ts        # getCart(), addToCart(), removeFromCart()
    ├── order.service.ts       # placeOrder(), getOrders(), getOrderById()
    └── user.service.ts        # getUser(), updateUser()
```

### Rules

- `lib/api/client.ts` — the only place that sets `Authorization` headers or reads `process.env` API keys
- `lib/api/endpoints.ts` — all external API URLs as `UPPER_CASE` constants (`ENDPOINTS.PRODUCTS`, `ENDPOINTS.AUTH_LOGIN`)
- `lib/services/` — pure async functions, no React, no Redux; can be called from RSCs or Route Handlers
- `app/api/*/route.ts` — Route Handlers call `lib/services/` then return `NextResponse.json()`
- Client components call `/api/*` via `fetch` or RTK Query — never `lib/services/` directly
- Never call external APIs from `'use client'` components directly

---

## State Management — Redux Toolkit

> Based on the [official RTK + Next.js App Router guide](https://redux-toolkit.js.org/usage/nextjs).

### Core rules (enforced by RTK docs)

- **No global store** — use `makeStore()` (a factory function), not `configureStore()` exported as a singleton. This prevents cross-request state leakage on the server.
- **RSCs never touch Redux** — Server Components cannot use hooks or context. They fetch data via `lib/services/` and pass it as props.
- **Redux is for globally shared, mutable client state only** — e.g. auth session, cart contents, UI state (mobile menu open, active modal). Do not put server-fetched data in Redux.
- **RTK Query** — use for client-side remote data fetching (polling, optimistic updates). Server-side fetching uses async RSCs with `fetch`.
- **`StoreProvider`** is a `'use client'` component placed in the root `layout.tsx` — available to all route groups. `cartSlice` (Navbar cart icon) and `uiSlice` (mobile menu) are needed on public and auth pages too.

### Folder structure

```
lib/redux/
├── store.ts                   # makeStore() factory — NOT a global variable
├── hooks.ts                   # useAppDispatch, useAppSelector, useAppStore (typed)
├── StoreProvider.tsx          # 'use client' — useRef guard so store is created once
└── slices/
    ├── authSlice.ts           # User session (id, name, token) — globally shared
    ├── cartSlice.ts           # Cart items, quantities — persists across pages
    └── uiSlice.ts             # Mobile menu open/closed, active modal, toast queue

app/
└── layout.tsx                 # Root layout — wraps <StoreProvider>
```

### `lib/redux/store.ts` pattern

```ts
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "./slices/authSlice";
import cartReducer from "./slices/cartSlice";
import uiReducer from "./slices/uiSlice";

export const makeStore = () =>
  configureStore({
    reducer: { auth: authReducer, cart: cartReducer, ui: uiReducer },
  });

export type AppStore = ReturnType<typeof makeStore>;
export type RootState = ReturnType<AppStore["getState"]>;
export type AppDispatch = AppStore["dispatch"];
```

### `lib/redux/StoreProvider.tsx` pattern

```tsx
"use client";
import { useRef } from "react";
import { Provider } from "react-redux";
import { makeStore, AppStore } from "./store";

export default function StoreProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  const storeRef = useRef<AppStore | null>(null);
  if (!storeRef.current) storeRef.current = makeStore();
  return <Provider store={storeRef.current}>{children}</Provider>;
}
```

### What goes in Redux vs where else

| Data                                  | Where                                              |
| ------------------------------------- | -------------------------------------------------- |
| Auth session (user id, role)          | Redux `authSlice`                                  |
| Cart items and quantities             | Redux `cartSlice`                                  |
| Mobile menu open/closed, active modal | Redux `uiSlice`                                    |
| Product catalogue, product detail     | RSC via `lib/services/` — server-fetched, no Redux |
| Order history                         | RSC via `lib/services/` — server-fetched, no Redux |
| Client-side remote data with polling  | RTK Query                                          |
| Form state (checkout, login)          | React Hook Form (local)                            |
| URL state (filters, pagination)       | `useSearchParams`                                  |

---

## Component Architecture — Atomic Design

All components follow **Atomic Design** methodology. Every new component must be placed at the correct level — never skip levels or mix concerns.

| Level         | Folder                  | Description                                                                                         | Examples                                                                                     |
| ------------- | ----------------------- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Atoms**     | `components/atoms/`     | Smallest indivisible UI primitives. No dependencies on other components. Pure token-driven styling. | Button, Badge, Input, Label, Icon, Avatar, Spinner, PriceTag, Rating                         |
| **Molecules** | `components/molecules/` | Combinations of atoms that form a simple, reusable unit with a single responsibility.               | FormField (Label + Input), SearchBar (Input + Button), CartItem (Image + PriceTag)           |
| **Organisms** | `components/organisms/` | Complex UI sections composed of molecules and/or atoms. Can hold local state.                       | Navbar (Logo + Nav + Cart icon), ProductCard (Image + Badge + PriceTag + Button), CartDrawer |
| **Templates** | `components/templates/` | Page-level layout skeletons — define structure with slots/children, no real data.                   | StorefrontLayout, AuthLayout, AccountLayout, CheckoutLayout                                  |
| **Pages**     | `app/**/page.tsx`       | Next.js App Router route files. Fill templates with real data. Do not contain UI logic.             | `app/(public)/products/page.tsx`, `app/(account)/account/orders/page.tsx`                    |

### Rules

- Atoms must **only** use design tokens (Tailwind utilities from `tokens.css`) — no hardcoded values
- Molecules import only atoms
- Organisms import molecules and/or atoms
- Templates import organisms and define layout — accept all content via props/children
- Pages are Next.js route files only — delegate all rendering to templates/organisms
- Every component at every level gets a co-located `.stories.tsx` file — only create these after Storybook is installed and confirmed running (`bun run storybook` succeeds). Never create story files before verifying Storybook works.
- **Always use `<Link>` from `'next/link'` for internal navigation — never a bare `<a>` tag.**
- **Always use `<Image>` from `'next/image'` for images — never `<img>`.**
- **Before writing any component that uses navigation, images, fonts, or metadata: read the relevant guide in `node_modules/next/dist/docs/` first.**

---

## MCP Servers — Reference

All six servers are configured in `.vscode/mcp.json`. VS Code is the only client. See `SETUP.md` for installation steps.

| Server                    | Type                                | Tools                                                                                                                                                                                                            |
| ------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `next-devtools`           | official (npx)                      | `get_errors`, `get_logs`, `get_routes`, `get_page_metadata`, `get_project_metadata`, `get_server_action_by_id`                                                                                                   |
| `storybook`               | official (npx)                      | `write_story`, `get_stories`                                                                                                                                                                                     |
| `tw-maker`                | custom (`bun run mcp/index.ts`)     | `list_components`, `read_component`, `write_story`, `get_design_tokens`                                                                                                                                          |
| `figma`                   | community (node_modules, sandboxed) | reads Figma file data, component specs, design tokens, styles                                                                                                                                                    |
| `design-system-extractor` | community (node_modules, sandboxed) | extracts component HTML, computed styles, props, dependencies, theme tokens from live Storybook                                                                                                                  |
| `lighthouse`              | community (node_modules, sandboxed) | `run_audit`, `get_performance_score`, `get_core_web_vitals`, `get_accessibility_score`, `get_seo_analysis`, `get_security_audit`, `find_unused_javascript`, `compare_mobile_desktop`, `check_performance_budget` |

**Which server to use:**
| Need | Server |
|---|---|
| "What errors does my app have?" | `next-devtools` → `get_errors` |
| "What routes exist?" | `next-devtools` → `get_routes` |
| "List all my components" | `tw-maker` → `list_components` |
| "Read the Button props" | `tw-maker` → `read_component` |
| "Write a story for Navbar" | `storybook` MCP first, fallback to `tw-maker` → `write_story` |
| "Implement this Figma design" | `figma` → read node data → generate component |
| "What tokens does Button use?" | `design-system-extractor` → extract from live Storybook |
| "What's the Lighthouse score?" | `lighthouse` → `get_performance_score` |
| "Are there accessibility issues?" | `lighthouse` → `get_accessibility_score` |
| "Check Core Web Vitals" | `lighthouse` → `get_core_web_vitals` |

---

## TypeScript — Configuration & Best Practices

### Current `tsconfig.json` settings (already configured)

- `strict: true` — enables all strict checks (`noImplicitAny`, `strictNullChecks`, etc.)
- `moduleResolution: bundler` — correct for Next.js + Bun
- `isolatedModules: true` — required for SWC/Turbopack transpilation
- `paths: { "@/*": ["./src/*"] }` — use `@/` for all absolute imports from `src/`

### Enable typed routes (add to `next.config.ts`)

```ts
import type { NextConfig } from "next";
const nextConfig: NextConfig = {
  typedRoutes: true, // Next.js validates all href/push/replace strings at compile time
};
export default nextConfig;
```

### IDE setup (required once per machine)

In VS Code: `Cmd+Shift+P` → "TypeScript: Select TypeScript Version" → **Use Workspace Version**
This activates the Next.js TypeScript plugin which warns on invalid segment config options, misplaced `'use client'`, and hooks-in-server-components errors.

### Type placement strategy

**Default: inline.** Types live in the file that owns them. Extract to `lib/types/` only when a type is shared across multiple files.

| Situation                                 | Where the type lives                            |
| ----------------------------------------- | ----------------------------------------------- |
| Simple component props (≤ 10 lines)       | Inline in `Button.tsx`                          |
| Complex component with many prop variants | `Button/Button.types.ts` next to `Button.tsx`   |
| Domain types shared across lib/           | `lib/types/auth.types.ts`                       |
| API DTOs shared across services/routes    | `lib/types/auth.api.types.ts`                   |
| Slice state type                          | Inline in `authSlice.ts` — private to the slice |
| Env var declarations                      | `global.d.ts` at project root — ambient only    |

### Shared types structure

```
lib/types/
├── auth.types.ts      # User, Session, Role, AuthState
├── auth.api.types.ts  # LoginRequest, LoginResponse, UserResponse, ApiError
└── cart.types.ts      # CartItem, CartState
```

### Importing shared types

```ts
// components/organisms/Navbar/Navbar.tsx
import type { User } from "@/lib/types/auth.types";

// app/api/auth/route.ts
import type { LoginRequest, LoginResponse } from "@/lib/types/auth.api.types";

// lib/services/auth.service.ts
import type { LoginRequest, LoginResponse } from "@/lib/types/auth.api.types";
```

`lib/types/` is the single source of truth for all shared domain types.

### `RootState` and `AppDispatch` — inferred, never declared manually

```ts
// lib/redux/store.ts
export const makeStore = () =>
  configureStore({ reducer: { auth: authReducer, ui: uiReducer } });
export type AppStore = ReturnType<typeof makeStore>;
export type RootState = ReturnType<AppStore["getState"]>; // inferred
export type AppDispatch = AppStore["dispatch"]; // inferred
```

> Never modify `next-env.d.ts` — it is auto-generated and will be overwritten. Put custom ambient types in `global.d.ts` at the project root and add it to `tsconfig.json` `include`:

```json
"include": ["global.d.ts", "next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/dev/types/**/*.ts"]
```

### Naming conventions

| Construct                      | Convention                                              | Example                                    |
| ------------------------------ | ------------------------------------------------------- | ------------------------------------------ |
| Types & Interfaces             | `PascalCase`                                            | `UserProfile`, `ButtonProps`               |
| Prefer `type` over `interface` | Use `interface` only when declaration merging is needed | `type ButtonProps = {...}`                 |
| Generic type params            | Single uppercase letter or descriptive `TPascal`        | `T`, `TData`, `TError`                     |
| Enums                          | Avoid — use `as const` objects instead                  | `const Role = { Admin: 'admin' } as const` |
| Props                          | Always suffix with `Props`                              | `NavbarProps`, `FormFieldProps`            |

### Patterns per layer

**Simple component props** — inline in the component file, nothing else imports them.

**Shared domain types** — put in `lib/types/` when used by more than one file:

- `lib/types/auth.types.ts` — domain types: `Role`, `User`, `Session`, `AuthState`
- `lib/types/auth.api.types.ts` — DTOs: `LoginRequest`, `LoginResponse`, `UserResponse`, `ApiError`
- `lib/types/cart.types.ts` — `CartItem`, `CartState`

**Next.js route files** — use built-in helpers, not custom types:

- `PageProps` and `LayoutProps` from `'next'` — do not write your own
- RSC `async` components are typed automatically by TypeScript 5+ — no `FC` wrapper needed

**API service and Route Handler** — both import from `lib/types/`:

- Service: `import type { LoginRequest, LoginResponse } from '@/lib/types/auth.api.types'`
- Route Handler: `import type { LoginRequest, LoginResponse } from '@/lib/types/auth.api.types'`

**Redux slice** — imports domain types from `lib/types/`, state type inline in slice.

**Environment variables** — ambient declaration in `global.d.ts` at project root:

```ts
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_API_URL: string;
    API_SECRET_KEY: string;
  }
}
```

### Rules

- **Inline by default** — props and simple types stay in the file that owns them
- **`lib/types/` for shared types** — only when a type is imported by more than one file
- **No feature-nested `types/` subfolders** — `lib/types/` is the single flat source of truth
- **Slice state types inline** — private to the slice file, not exported to `lib/types/`
- **API DTOs in `lib/types/`** — Route Handlers and services both import from `@/lib/types/`
- **Only `global.d.ts` at root** — solely for ambient declarations (`NodeJS.ProcessEnv`, module augmentations)
- Never use `any` — use `unknown` and narrow with type guards instead
- Never use non-null assertion (`!`) without a comment explaining why it is safe
- RSC `async` components are typed automatically by TypeScript 5+ — no extra `FC` wrapper needed
- Keep `next-env.d.ts` in `.gitignore` (auto-generated on `next dev`/`next build`)

---

## Key Decisions

- **Atomic Design** — strict 5-level hierarchy (atoms → molecules → organisms → templates → pages); each level only imports from levels below it
- **Route Groups** — `(public)`, `(auth)`, `(account)` give each section its own layout without affecting URLs
- **Public pages are Server Components** — the Header reads session server-side to render guest vs authenticated nav; no Redux needed on public pages
- **SEO via metadata API** — `title.template` in root layout, static `metadata` for fixed pages, `generateMetadata` for dynamic pages (products, categories); never `<meta>` in JSX
- **GTM & trackers in `lib/analytics/`** — `GTMScript` + `GTMNoScript` components, loaded via `<Script strategy="afterInteractive">` in root layout; container ID in `NEXT_PUBLIC_GTM_ID`
- **Auth redirects via `proxy.ts`** — Next.js 16 renamed Middleware to Proxy; `src/proxy.ts` (inside `src/`) handles `(auth)` redirects (logged-in → `/account`) and `(account)` redirects (guest → `/login`)
- **BFF proxy** — client never calls external APIs directly; Route Handlers in `app/api/` proxy all external calls so secrets stay server-side
- **RTK `makeStore` not singleton** — per Next.js App Router requirements; prevents cross-request state contamination
- **Redux for mutable global state only** — auth session + cart + UI state; server data (products, orders) stays in RSC props or RTK Query
- **Cart in Redux** — cart state is client-side, mutable, and shared across pages (Navbar icon, cart page, checkout); it belongs in `cartSlice`
- **Products and orders are server-fetched** — product catalogue and order history are read-only, fetched in RSCs via `lib/services/`, never stored in Redux
- `tokens.json` over YAML — no extra dependency, simple `JSON.parse/stringify`
- CSS generator script (build-time) over runtime import — no runtime overhead
- `tokens.css` is a **generated file** — isolated, optionally `.gitignore`-able
- **Six MCP servers**: `next-devtools-mcp`, `design-system` (custom), `storybook`, `figma`, `design-system-extractor`, `lighthouse` — all in `.vscode/mcp.json` (single file, VS Code is the only client)
- **MCP security**: official packages (`next-devtools`, `storybook`) use `npx @latest`; community packages (`figma`, `design-system-extractor`, `lighthouse`) installed as dev deps with pinned versions, run from `node_modules`, sandboxed via `sandboxEnabled: true`
- MCP is **tools-only, no embedded LLM** — the AI assistant calling the tools supplies intelligence; no API key required

## Directory Structure (target)

```
tw-maker/
├── src/
│   ├── app/
│   │   ├── (public)/
│   │   │   ├── layout.tsx            # Public layout — Header (auth-aware) + Footer
│   │   │   ├── page.tsx              # / (home — hero, featured products)
│   │   │   ├── products/
│   │   │   │   ├── page.tsx          # /products (catalogue with filters)
│   │   │   │   └── [slug]/
│   │   │   │       └── page.tsx      # /products/[slug] (product detail)
│   │   │   ├── cart/
│   │   │   │   └── page.tsx          # /cart
│   │   │   ├── checkout/
│   │   │   │   └── page.tsx          # /checkout
│   │   │   └── about/
│   │   │       └── page.tsx          # /about
│   │   ├── (auth)/
│   │   │   ├── layout.tsx            # Auth layout — centered card, no header/footer
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── signup/
│   │   │       └── page.tsx
│   │   ├── (account)/
│   │   │   ├── layout.tsx            # Account layout — auth-gated
│   │   │   ├── loading.tsx
│   │   │   └── account/
│   │   │       ├── page.tsx          # /account (profile)
│   │   │       ├── orders/
│   │   │       │   ├── page.tsx      # /account/orders (order history)
│   │   │       │   └── [id]/
│   │   │       │       └── page.tsx  # /account/orders/[id] (order detail)
│   │   │       └── settings/
│   │   │           └── page.tsx      # /account/settings
│   │   ├── api/
│   │   │   └── [feature]/
│   │   │       └── route.ts          # BFF Route Handlers — proxy to external APIs
│   │   ├── favicon.ico               # Auto-detected by Next.js
│   │   ├── opengraph-image.tsx       # Default OG image (ImageResponse)
│   │   ├── robots.ts                 # Generates /robots.txt
│   │   ├── sitemap.ts                # Generates /sitemap.xml
│   │   ├── globals.css
│   │   ├── tokens.css                # GENERATED — do not edit manually
│   │   └── layout.tsx                # Root layout — wraps <StoreProvider>, fonts, metadata
│   ├── lib/
│   │   ├── redux/
│   │   │   ├── store.ts              # makeStore() — NOT a global
│   │   │   ├── hooks.ts              # useAppDispatch, useAppSelector, useAppStore
│   │   │   ├── StoreProvider.tsx     # 'use client' Redux provider
│   │   │   └── slices/
│   │   │       ├── authSlice.ts      # User session (id, name, token)
│   │   │       ├── cartSlice.ts      # CartItem[], quantities — persists across pages
│   │   │       └── uiSlice.ts        # Mobile menu, active modal, toast queue
│   │   ├── api/
│   │   │   ├── client.ts             # axios instance — baseURL, auth headers, interceptors
│   │   │   └── endpoints.ts          # All external API URLs as UPPER_CASE constants
│   │   ├── services/
│   │   │   ├── auth.service.ts       # login(), signup(), refreshToken()
│   │   │   ├── product.service.ts    # getProducts(), getProductBySlug()
│   │   │   ├── cart.service.ts       # getCart(), addToCart(), removeFromCart()
│   │   │   ├── order.service.ts      # placeOrder(), getOrders(), getOrderById()
│   │   │   └── user.service.ts       # getUser(), updateUser()
│   │   ├── analytics/
│   │   │   ├── GTMScript.tsx         # <Script strategy="afterInteractive"> for GTM
│   │   │   ├── GTMNoScript.tsx       # <noscript> iframe fallback
│   │   │   └── index.ts              # re-exports all tracker components
│   │   ├── seo/
│   │   │   ├── defaults.ts           # SITE_NAME, SITE_URL, default OG image
│   │   │   └── helpers.ts            # buildProductMeta(), buildCategoryMeta()
│   │   └── types/
│   │       ├── auth.types.ts         # User, Session, Role, AuthState
│   │       ├── auth.api.types.ts     # LoginRequest/Response, UserResponse, ApiError
│   │       └── cart.types.ts         # CartItem, CartState
│   ├── components/
│   │   ├── atoms/
│   │   │   ├── Button/
│   │   │   │   ├── Button.tsx
│   │   │   │   └── Button.stories.tsx
│   │   │   ├── Badge/
│   │   │   │   ├── Badge.tsx
│   │   │   │   └── Badge.stories.tsx
│   │   │   ├── Input/
│   │   │   │   ├── Input.tsx
│   │   │   │   └── Input.stories.tsx
│   │   │   ├── Label/
│   │   │   │   ├── Label.tsx
│   │   │   │   └── Label.stories.tsx
│   │   │   ├── Icon/
│   │   │   │   ├── Icon.tsx
│   │   │   │   └── Icon.stories.tsx
│   │   │   ├── Avatar/
│   │   │   │   ├── Avatar.tsx
│   │   │   │   └── Avatar.stories.tsx
│   │   │   └── Spinner/
│   │   │       ├── Spinner.tsx
│   │   │       └── Spinner.stories.tsx
│   │   ├── molecules/
│   │   │   ├── FormField/
│   │   │   │   ├── FormField.tsx
│   │   │   │   └── FormField.stories.tsx
│   │   │   └── SearchBar/
│   │   │       ├── SearchBar.tsx
│   │   │       └── SearchBar.stories.tsx
│   │   ├── organisms/
│   │   │   ├── Navbar/
│   │   │   │   ├── Navbar.tsx
│   │   │   │   └── Navbar.stories.tsx
│   │   │   └── ProductCard/
│   │   │       ├── ProductCard.tsx
│   │   │       └── ProductCard.stories.tsx
│   │   └── templates/
│   │       ├── StorefrontLayout/
│   │       │   ├── StorefrontLayout.tsx
│   │       │   └── StorefrontLayout.stories.tsx
│   │       ├── AuthLayout/
│   │       │   ├── AuthLayout.tsx
│   │       │   └── AuthLayout.stories.tsx
│   │       ├── AccountLayout/
│   │       │   ├── AccountLayout.tsx
│   │       │   └── AccountLayout.stories.tsx
│   │       └── CheckoutLayout/
│   │           ├── CheckoutLayout.tsx
│   │           └── CheckoutLayout.stories.tsx
│   ├── mcp/
│   │   ├── index.ts
│   │   └── tools/
│   │       ├── list_components.ts
│   │       ├── read_component.ts
│   │       ├── write_story.ts
│   │       └── get_design_tokens.ts
│   ├── scripts/
│   │   └── generate-tokens-css.ts
│   ├── tokens/
│   │   └── tokens.json               # SOURCE OF TRUTH — all design tokens
│   └── proxy.ts                      # Auth redirects — Next.js 16 replaces middleware.ts
├── global.d.ts                       # Ambient: NodeJS.ProcessEnv, module augmentations
├── .storybook/
│   ├── main.ts
│   └── preview.ts
└── .vscode/
    └── mcp.json                      # ALL MCP servers (single source of truth)
```
