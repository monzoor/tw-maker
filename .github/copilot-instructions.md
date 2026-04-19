# Copilot Workspace Instructions

> **These rules are always active for every Copilot conversation in this workspace.**
> They govern architecture, conventions, and how features should be built.
> For one-time project initialization, use `SETUP.md` instead — do not re-run bootstrap or setup steps unless explicitly instructed.

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
> ⚠️ **Cross-verification is mandatory.** Whenever there is any doubt about how a Next.js feature behaves — including APIs, file conventions, caching, rendering modes, or config options — you must read the corresponding guide in `node_modules/next/dist/docs/` before proceeding. Training data and online resources must never override the bundled docs.

---

## App Routing Architecture

Routes are organised with **Next.js Route Groups** (`(folderName)`) so each section gets its own layout without polluting the URL.

| Route Group | URL prefix                                          | Layout                          | Who can access              |
| ----------- | --------------------------------------------------- | ------------------------------- | --------------------------- |
| `(public)`  | `/`, `/products`, `/products/[slug]`, `/cart`, etc. | Header (auth-aware) + Footer    | Everyone — logged in or not |
| `(auth)`    | `/login`, `/signup`                                 | Centered card, no header/footer | Unauthenticated users only  |
| `(account)` | `/account`, `/account/orders`, `/account/settings`  | Header + account nav            | Authenticated users only    |

- `(public)` pages are Server Components that render for all visitors. The Header reads the session server-side and conditionally renders a guest nav (Login / Sign up) or an authenticated nav (avatar, account link).
- `(auth)` pages redirect to `/account` if the user is already logged in.
- `(account)` pages redirect to `/login` if the user is not logged in. `proxy.ts` handles both redirects.
- `StoreProvider` wraps the root `layout.tsx` — all route groups (public, auth, account) have access to Redux state. `cartSlice` and `uiSlice` are needed on public pages (Navbar cart icon, mobile menu), so scoping the store to `(account)` only would break those.

---

## Server and Client Components

> Before writing any component, read `node_modules/next/dist/docs/01-app/01-getting-started/05-server-and-client-components.md`.

All components are **Server Components by default** — no directive is needed. Only add `'use client'` when a component requires client-only features.

### Rules

- **Server Component is the default** — never add `'use client'` unless one of the conditions above applies; omitting it reduces the JS bundle
- **Push `'use client'` as deep as possible** — marking a large layout as a Client Component pulls all its children into the client bundle; extract only the interactive leaf
- **`'use client'` is a boundary, not a per-file flag** — once a file has the directive, all its imports and child components are part of the client bundle; you do not repeat it for every file in the subtree
- **Server Components cannot use hooks or context** — pass data as props from the Server Component to a Client Component
- **Props crossing the Server→Client boundary must be serializable** — no functions, class instances, Promises, Maps, or Sets
- **Wrap third-party components that lack `'use client'`** — re-export them from a local file that adds the directive
- **Mark server-only modules with `import 'server-only'`** — add it to `lib/api/client.ts` and all `lib/services/*.ts` files; this causes a build-time error if they are accidentally imported in a Client Component

---

## Error Handling

> Before writing any error UI, read `node_modules/next/dist/docs/01-app/01-getting-started/10-error-handling.md`.

Next.js splits errors into two categories: **expected errors** (failed API calls, validation) and **uncaught exceptions** (bugs, crashes). Each is handled differently.

### Expected errors

- **Server Components** — check the response and return an error UI or call `redirect()`. Do not `throw`.
- **Route Handlers** — return `NextResponse.json({ error: '...' }, { status: 4xx })` and handle in the client.
- **Server Functions (mutations)** — model errors as return values (e.g. `{ message: 'Failed' }`), read with `useActionState`. Do not `throw` expected errors.

### Uncaught exceptions — single `error.tsx` at app root

This project uses **one** `error.tsx` at `src/app/error.tsx`. It wraps all route segments below the root layout and serves as the single 500-style fallback for the entire app. Must be a Client Component (`'use client'`).

**Key facts:**

- `error.tsx` does **not** catch errors in the `layout.tsx` at the same level — only `page.tsx` and nested layouts/pages below it
- `error.digest` contains a hash matching the server-side log entry — use it in error reporting
- `unstable_retry()` re-fetches and re-renders the segment; prefer it over the legacy `reset()`
- In production, `error.message` from Server Components is replaced with a generic string

### Root layout crashes — `global-error.tsx`

`global-error.tsx` is **not deprecated** but is rare — it only activates when the root `layout.tsx` itself crashes. It must render its own `<html>` and `<body>` tags and cannot use `metadata`/`generateMetadata`.

### 404 — single `not-found.tsx` at app root + `notFound()`

This project uses **one** `not-found.tsx` at `src/app/not-found.tsx`. Call `notFound()` from `'next/navigation'` anywhere in the segment tree — it bubbles up to the nearest (here, the root) `not-found.tsx`.

### Placement rules for this project

| File               | Where      | Catches                               |
| ------------------ | ---------- | ------------------------------------- |
| `error.tsx`        | `src/app/` | All runtime errors across every route |
| `not-found.tsx`    | `src/app/` | All `notFound()` calls — app-wide 404 |
| `global-error.tsx` | `src/app/` | Root layout crashes only (rare)       |

### Rules

- `error.tsx` must always be a Client Component — add `'use client'` at the top
- Always include `unstable_retry` — gives users a way to recover without a full page reload
- Log `error.digest` to your error reporting service (Sentry, Datadog) — it links to the server-side stack trace
- Never expose raw `error.message` in production UI for Server Component errors — use a generic message
- `not-found.tsx` must use `<Link>` from `'next/link'` for navigation — never a bare `<a>`
- `global-error.tsx` must include `<html>` and `<body>` — it replaces the root layout
- Do **not** create per-segment `error.tsx` or `not-found.tsx` files — the single root-level files handle everything
- Before modifying any error file, read `node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/error.md`

---

## Proxy (`proxy.ts`)

> ⚠️ **Next.js 16 renamed Middleware to Proxy.** The file is `src/proxy.ts` (inside `src/`, same level as `src/app/`). The function is named `proxy` (not `middleware`). The `middleware.ts` file convention is deprecated. Always use `proxy.ts`.

`proxy.ts` runs server-side before a request reaches any route. For this e-commerce site it handles two auth redirects:

- **`(auth)` guard** — `/login` and `/signup` redirect to `/account` if the user is already logged in
- **`(account)` guard** — all `/account/*` routes redirect to `/login` if the user is not logged in

**Rules:**

- `proxy.ts` must be inside `src/` (at `src/proxy.ts`) — same level as `src/app/`, never inside `app/` itself
- Export the function as `proxy` (named) or as `default` — not `middleware`
- `config.matcher` must be a static constant — no dynamic values
- Do not do slow data fetching inside `proxy.ts` — only fast cookie/header checks
- Pass data to the app via headers, cookies, or URL — not via shared globals
- Before modifying `proxy.ts`, read `node_modules/next/dist/docs/01-app/01-getting-started/16-proxy.md`

---

## Fonts

> Before adding fonts, read `node_modules/next/dist/docs/01-app/01-getting-started/13-fonts.md`.

Use `next/font` — never load fonts via a `<link>` tag in JSX or from an external CDN. `next/font` self-hosts all fonts, removes external network requests, and eliminates layout shift (zero CLS).

### Rules

- Use `next/font/google` for Google Fonts and `next/font/local` for self-hosted files — never `<link rel="stylesheet">` pointing to an external CDN
- Initialize the font module at the top level of `src/app/layout.tsx` (outside the component function) and apply the returned `className` to `<html>` for app-wide coverage
- Always set `lang="en"` (or the appropriate locale) on the `<html>` element in the root layout — Next.js does not add it automatically; it is required for accessibility (WCAG 3.1.1) and is an SEO signal
- Specify `subsets` to include only the character sets the design system uses — keeps font payloads small
- For variable fonts use a `weight` range; for non-variable fonts list only the specific weights used in `tokens.json`
- Do not initialize the same font in multiple files — define it once in root layout; expose it as a CSS variable if components need to reference it
- Read `node_modules/next/dist/docs/01-app/03-api-reference/02-components/font.md` for the full options reference

## Data Fetching — BFF

The browser **never** calls external APIs directly. All external calls go through a **BFF (Backend for Frontend)** proxy using Next.js Route Handlers. Secrets (API keys, tokens) stay server-side.

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
- **Serializable state only** — never put Promises, class instances, functions, Maps, Sets, or Symbols in state or dispatched actions. This breaks Redux DevTools time-travel and causes silent bugs. RTK's `configureStore` throws a warning in development when non-serializable values are detected.
- **RTK Query** — use for client-side remote data fetching (polling, optimistic updates). Server-side fetching uses async RSCs with `fetch`.
- **`StoreProvider`** is a `'use client'` component placed in the root `layout.tsx` — available to all route groups. `cartSlice` (Navbar cart icon) and `uiSlice` (mobile menu) are needed on public and auth pages too.
- **Dynamic routes that show user-specific data** must opt out of Next.js route caching: add `export const dynamic = 'force-dynamic'` to the page file. After any server mutation (place order, update profile), call `revalidatePath()` or `revalidateTag()` so the cache is invalidated and the next RSC fetch returns fresh data.

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

### `lib/redux/store.ts`

`store.ts` exports `makeStore` — a factory function that calls `configureStore()` with `authReducer`, `cartReducer`, and `uiReducer`. Export `AppStore` as the return type of `makeStore`, then infer `RootState` and `AppDispatch` from `AppStore` — never declare them manually.

### `lib/redux/StoreProvider.tsx`

`StoreProvider` is a `'use client'` component. It holds the store instance in `useRef` (created once via `makeStore()` on first render) and wraps `children` in the react-redux `<Provider>`. The `useRef` guard prevents a new store being created on every render.

### What goes in Redux vs where else

| Data                                  | Where                                              |
| ------------------------------------- | -------------------------------------------------- |
| Auth session (user id, role)          | Redux `authSlice`                                  |
| Cart items and quantities             | Redux `cartSlice`                                  |
| Mobile menu open/closed, active modal | Redux `uiSlice`                                    |
| Product catalogue, product detail     | RSC via `lib/services/` — server-fetched, no Redux |
| Order history                         | RSC via `lib/services/` — server-fetched, no Redux |
| Client-side remote data with polling  | RTK Query                                          |
| URL state (filters, pagination)       | `useSearchParams`                                  |

---

### `createAsyncThunk` — async actions with loading states

Use `createAsyncThunk` for any async operation that needs to be tracked in the store with `pending` / `fulfilled` / `rejected` states — e.g. user login, cart sync to server, profile update. Define thunks **in the same slice file** that owns the state they affect.

Every async slice has four state fields: the domain data (e.g. `user`, `token`), `status: 'idle' | 'pending' | 'fulfilled' | 'rejected'`, and `error: string | null`. Always type all three `createAsyncThunk` generics — fulfilled payload, argument, and `{ rejectValue: string }`. Wire `pending`, `fulfilled`, and `rejected` cases in `extraReducers` using the builder API.

**Thunk rules:**

- Define thunks in the same slice file that owns the affected state — never in a separate file
- Always type the three generics: `<FulfilledPayload, Argument, { rejectValue: string }>`
- Use `rejectWithValue()` for user-facing error messages — never `throw` raw errors from a thunk
- The `status` field on each async slice should use the literal union `'idle' | 'pending' | 'fulfilled' | 'rejected'` — never a boolean `isLoading` flag
- Do not store the serialized `Error` object in state — extract the message string instead
- Every slice that owns async state **must** expose a `reset` reducer that returns the slice to its initial state — call it before re-triggering a thunk and in cleanup effects
- Dispatch thunks from domain hooks in `src/hooks/`, never directly from components

---

### Selectors — read state with named `select*` functions

Always read store state via named selector functions co-located in the slice file. Simple field reads are plain functions (`(state: RootState) => state.cart.items`). Use `createSelector` from RTK when the result requires computation or returns a new array/object — this memoises the output and prevents unnecessary re-renders.

**Selector rules:**

- Always prefix selector names with `select` — e.g. `selectCartItems`, `selectIsLoggedIn`
- Co-locate selectors in the slice file that owns the data
- Use `createSelector` when the derived value is a non-trivial computation or returns a new array/object (avoids referential equality failures)
- Never write inline `(state) => state.x.y` lambdas in `useAppSelector` calls — always call a named selector
- Simple field reads don't need `createSelector` — reserve it for filtered lists, totals, and object compositions

---

### Action naming — events, not setters

Model actions as **events that happened**, not as imperative setter commands. `createSlice` generates action types in the `"domain/eventName"` format — name reducers as past-tense events:

| ✅ Use                | ❌ Avoid           |
| --------------------- | ------------------ |
| `cart/itemAdded`      | `cart/setItems`    |
| `cart/itemRemoved`    | `cart/setQuantity` |
| `auth/sessionCleared` | `auth/setUser`     |
| `ui/mobileMenuOpened` | `ui/setMenuOpen`   |

---

## Custom Hooks

All `useState`, `useEffect`, `useSelector`, `useDispatch`, and any other hook calls must live in a dedicated hook file — never directly in a `.tsx` component or page file. Components are pure rendering; hooks own all state and side-effect logic.

### Placement rule

All hooks live in `src/hooks/` — flat, no sub-folders, one file per logical concern. There is no co-location of hook files inside component folders.

### Types

Return types are defined **inline** in the hook file and exported from it. Never create a separate `.types.ts` for hook types — import `UseXxxReturn` directly from the hook file if another file needs the type.

### Naming conventions

| Construct           | Convention                 | Example                            |
| ------------------- | -------------------------- | ---------------------------------- |
| Hook file           | `use` + domain, camelCase  | `useCart.ts`, `useNavbar.ts`       |
| Hook function       | same as file name          | `export function useCart()`        |
| Return type         | `Use` + domain + `Return`  | `UseCartReturn`, `UseNavbarReturn` |
| Options/params type | `Use` + domain + `Options` | `UseCartOptions`                   |

### Rules

- Never call `useState`, `useEffect`, `useSelector`, `useDispatch`, `useRef`, `useCallback`, or `useMemo` directly in a `.tsx` file — wrap them in a hook in `src/hooks/`
- Every hook file starts with `'use client'` — hooks are always client-side
- `lib/redux/hooks.ts` is **exempt** — it holds only the three typed RTK wrappers (`useAppDispatch`, `useAppSelector`, `useAppStore`) and is not a domain hook
- Domain hooks (`useCart`, `useAuth`, etc.) call `useAppDispatch`/`useAppSelector` from `lib/redux/hooks` — never the raw RTK hooks
- Hook return types are inline in the hook file — move to `lib/types/` only if another file imports `UseXxxReturn` as a standalone prop type

---

## Component Architecture — Atomic Design

All components follow **Atomic Design** methodology. Every new component must be placed at the correct level — never skip levels or mix concerns.

| Level         | Folder                  | Description                                                                                                                                             | Examples                                                                                     |
| ------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Atoms**     | `components/atoms/`     | Smallest indivisible UI primitives. No dependencies on other components. Pure token-driven styling.                                                     | Button, Badge, Input, Label, Icon, Avatar, Spinner, PriceTag, Rating                         |
| **Molecules** | `components/molecules/` | Combinations of atoms that form a simple, reusable unit with a single responsibility.                                                                   | FormField (Label + Input), SearchBar (Input + Button), CartItem (Image + PriceTag)           |
| **Organisms** | `components/organisms/` | Complex UI sections composed of molecules and/or atoms. All state and effect logic is delegated to a hook in `src/hooks/` — never inline in the `.tsx`. | Navbar (Logo + Nav + Cart icon), ProductCard (Image + Badge + PriceTag + Button), CartDrawer |
| **Templates** | `components/templates/` | Page-level layout skeletons — define structure with slots/children, no real data.                                                                       | StorefrontLayout, AuthLayout, AccountLayout, CheckoutLayout                                  |
| **Pages**     | `app/**/page.tsx`       | Next.js App Router route files. Fill templates with real data. Do not contain UI logic.                                                                 | `app/(public)/products/page.tsx`, `app/(account)/account/orders/page.tsx`                    |

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

> The full component list with file paths is in the **Directory Structure** section below. Use that as the authoritative inventory — check it before creating new components to avoid duplicates.

---

## MCP Servers — Reference

All six servers are configured in `.vscode/mcp.json`. VS Code is the only client. See `SETUP.md` for installation steps.

| Server                    | Type                                | Tools                                                                                                                                                                                                            |
| ------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `next-devtools`           | official (npx)                      | `get_errors`, `get_logs`, `get_routes`, `get_page_metadata`, `get_project_metadata`, `get_server_action_by_id`                                                                                                   |
| `storybook`               | official (npx)                      | `write_story`, `get_stories`                                                                                                                                                                                     |
| `tw-maker`                | custom (`bun run src/mcp/index.ts`) | `list_components`, `read_component`, `write_story`, `get_design_tokens`                                                                                                                                          |
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

### Enable typed routes

Add `typedRoutes: true` to `next.config.ts` — Next.js validates all `href`/`push`/`replace` strings at compile time.

### Type placement strategy

**Default: inline.** Types live in the file that owns them. Extract to `lib/types/` only when a type is shared across multiple files.

| Situation                                 | Where the type lives                                                |
| ----------------------------------------- | ------------------------------------------------------------------- |
| Simple component props (≤ 10 lines)       | Inline in `Button.tsx`                                              |
| Complex component with many prop variants | `Button/Button.types.ts` next to `Button.tsx`                       |
| Domain types shared across lib/           | `lib/types/auth.types.ts`                                           |
| API DTOs shared across services/routes    | `lib/types/auth.api.types.ts`                                       |
| Slice state type                          | Inline in `authSlice.ts` — private to the slice                     |
| Hook return type                          | Inline in `useCart.ts` as `UseCartReturn` — exported from hook file |
| Env var declarations                      | `global.d.ts` at project root — ambient only                        |

### `RootState` and `AppDispatch` — inferred, never declared manually

Infer both types from the return type of `makeStore` (see `lib/redux/store.ts` pattern in the Redux section) — never write them manually.

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
│   │   ├── error.tsx                 # App-wide 500 fallback — must be 'use client'
│   │   ├── global-error.tsx          # Catches root layout crashes — must include <html>+<body>
│   │   ├── not-found.tsx             # App-wide 404 — unmatched routes
│   │   ├── styles/
│   │   │   ├── globals.css
│   │   │   └── tokens.css            # GENERATED — do not edit manually
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
│   ├── hooks/
│   │   ├── useAuth.ts                # reads authSlice — user, isLoggedIn, logout
│   │   ├── useCart.ts                # reads cartSlice — items, totalCount, add, remove
│   │   ├── useUI.ts                  # reads uiSlice — modal, drawer, toast helpers
│   │   ├── useNavbar.ts              # Navbar-specific logic
│   │   └── useProductFilters.ts      # /products filter state, pagination, URL sync
│   ├── components/
│   │   ├── atoms/
│   │   │   ├── Button/                   # Primary, secondary, ghost, destructive; sm/md/lg
│   │   │   │   ├── Button.tsx
│   │   │   │   └── Button.stories.tsx
│   │   │   ├── IconButton/               # Icon-only button with aria-label
│   │   │   │   ├── IconButton.tsx
│   │   │   │   └── IconButton.stories.tsx
│   │   │   ├── Link/                     # Styled next/link wrapper
│   │   │   │   ├── Link.tsx
│   │   │   │   └── Link.stories.tsx
│   │   │   ├── Badge/                    # New, Sale, Out of Stock chip
│   │   │   │   ├── Badge.tsx
│   │   │   │   └── Badge.stories.tsx
│   │   │   ├── Lozenge/                  # Pending, Shipped, Delivered, Refund
│   │   │   │   ├── Lozenge.tsx
│   │   │   │   └── Lozenge.stories.tsx
│   │   │   ├── Tag/                      # Category / filter pill
│   │   │   │   ├── Tag.tsx
│   │   │   │   └── Tag.stories.tsx
│   │   │   ├── PriceTag/                 # Formatted price, strikethrough variant
│   │   │   │   ├── PriceTag.tsx
│   │   │   │   └── PriceTag.stories.tsx
│   │   │   ├── DiscountBadge/            # Percentage-off callout
│   │   │   │   ├── DiscountBadge.tsx
│   │   │   │   └── DiscountBadge.stories.tsx
│   │   │   ├── ScarcityText/             # "Only 2 left" urgency text
│   │   │   │   ├── ScarcityText.tsx
│   │   │   │   └── ScarcityText.stories.tsx
│   │   │   ├── Input/                    # Text, email, number, search
│   │   │   │   ├── Input.tsx
│   │   │   │   └── Input.stories.tsx
│   │   │   ├── Textarea/                 # Multi-line input
│   │   │   │   ├── Textarea.tsx
│   │   │   │   └── Textarea.stories.tsx
│   │   │   ├── Checkbox/                 # With indeterminate state
│   │   │   │   ├── Checkbox.tsx
│   │   │   │   └── Checkbox.stories.tsx
│   │   │   ├── RadioButton/              # Single option selector
│   │   │   │   ├── RadioButton.tsx
│   │   │   │   └── RadioButton.stories.tsx
│   │   │   ├── Select/                   # Native dropdown
│   │   │   │   ├── Select.tsx
│   │   │   │   └── Select.stories.tsx
│   │   │   ├── Toggle/                   # Boolean switch
│   │   │   │   ├── Toggle.tsx
│   │   │   │   └── Toggle.stories.tsx
│   │   │   ├── QuantityInput/            # Number stepper with min/max
│   │   │   │   ├── QuantityInput.tsx
│   │   │   │   └── QuantityInput.stories.tsx
│   │   │   ├── Label/                    # Form field label
│   │   │   │   ├── Label.tsx
│   │   │   │   └── Label.stories.tsx
│   │   │   ├── Heading/                  # h1–h6 with size/weight variants
│   │   │   │   ├── Heading.tsx
│   │   │   │   └── Heading.stories.tsx
│   │   │   ├── Text/                     # Body copy, captions, helper text
│   │   │   │   ├── Text.tsx
│   │   │   │   └── Text.stories.tsx
│   │   │   ├── Icon/                     # SVG icon wrapper
│   │   │   │   ├── Icon.tsx
│   │   │   │   └── Icon.stories.tsx
│   │   │   ├── StarIcon/                 # Filled / half / empty star unit
│   │   │   │   ├── StarIcon.tsx
│   │   │   │   └── StarIcon.stories.tsx
│   │   │   ├── Avatar/                   # Profile image with fallback initials
│   │   │   │   ├── Avatar.tsx
│   │   │   │   └── Avatar.stories.tsx
│   │   │   ├── Spinner/                  # Loading indicator
│   │   │   │   ├── Spinner.tsx
│   │   │   │   └── Spinner.stories.tsx
│   │   │   ├── Skeleton/                 # Shimmer placeholder block
│   │   │   │   ├── Skeleton.tsx
│   │   │   │   └── Skeleton.stories.tsx
│   │   │   ├── ProgressBar/              # Checkout step / upload / free-shipping progress
│   │   │   │   ├── ProgressBar.tsx
│   │   │   │   └── ProgressBar.stories.tsx
│   │   │   ├── Rating/                   # Star display — read-only or interactive
│   │   │   │   ├── Rating.tsx
│   │   │   │   └── Rating.stories.tsx
│   │   │   ├── Dot/                      # Online indicator or unread count marker
│   │   │   │   ├── Dot.tsx
│   │   │   │   └── Dot.stories.tsx
│   │   │   ├── Divider/                  # Horizontal / vertical separator
│   │   │   │   ├── Divider.tsx
│   │   │   │   └── Divider.stories.tsx
│   │   │   ├── Spacer/                   # Fixed or flex gap utility
│   │   │   │   ├── Spacer.tsx
│   │   │   │   └── Spacer.stories.tsx
│   │   │   ├── Overlay/                  # Semi-transparent backdrop for modals/drawers
│   │   │   │   ├── Overlay.tsx
│   │   │   │   └── Overlay.stories.tsx
│   │   │   ├── Logo/                     # Brand mark — SVG or Image with Link
│   │   │   │   ├── Logo.tsx
│   │   │   │   └── Logo.stories.tsx
│   │   │   ├── CountdownTimer/           # Days/hours/minutes/seconds digits
│   │   │   │   ├── CountdownTimer.tsx
│   │   │   │   └── CountdownTimer.stories.tsx
│   │   │   ├── Tooltip/                  # Floating text on hover/focus
│   │   │   │   ├── Tooltip.tsx
│   │   │   │   └── Tooltip.stories.tsx
│   │   │   ├── Flag/                     # Country flag icon for locale/shipping
│   │   │   │   ├── Flag.tsx
│   │   │   │   └── Flag.stories.tsx
│   │   │   ├── Placeholder/              # Empty image fallback box
│   │   │   │   ├── Placeholder.tsx
│   │   │   │   └── Placeholder.stories.tsx
│   │   │   ├── AspectRatioBox/           # Enforces image aspect ratio — prevents CLS
│   │   │   │   ├── AspectRatioBox.tsx
│   │   │   │   └── AspectRatioBox.stories.tsx
│   │   │   ├── VideoPlayButton/          # Overlay play trigger for product videos
│   │   │   │   ├── VideoPlayButton.tsx
│   │   │   │   └── VideoPlayButton.stories.tsx
│   │   │   ├── ImageZoomCursor/          # Magnifier cursor indicator
│   │   │   │   ├── ImageZoomCursor.tsx
│   │   │   │   └── ImageZoomCursor.stories.tsx
│   │   │   ├── VisuallyHidden/           # Screen-reader-only text wrapper
│   │   │   │   ├── VisuallyHidden.tsx
│   │   │   │   └── VisuallyHidden.stories.tsx
│   │   │   ├── SkipLink/                 # "Skip to main content" — WCAG 2.4.1
│   │   │   │   ├── SkipLink.tsx
│   │   │   │   └── SkipLink.stories.tsx
│   │   │   └── LiveRegion/               # aria-live wrapper for dynamic announcements
│   │   │       ├── LiveRegion.tsx
│   │   │       └── LiveRegion.stories.tsx
│   │   ├── molecules/
│   │   │   ├── PriceDisplay/             # PriceTag + struck PriceTag + DiscountBadge
│   │   │   │   ├── PriceDisplay.tsx
│   │   │   │   └── PriceDisplay.stories.tsx
│   │   │   ├── RatingSummary/            # Rating stars + "4.2 / 5 · 128 reviews"
│   │   │   │   ├── RatingSummary.tsx
│   │   │   │   └── RatingSummary.stories.tsx
│   │   │   ├── ProductBadgeGroup/        # Stack of Sale / New / Low Stock badges
│   │   │   │   ├── ProductBadgeGroup.tsx
│   │   │   │   └── ProductBadgeGroup.stories.tsx
│   │   │   ├── StockIndicator/           # Dot + "In stock" / "Only 3 left"
│   │   │   │   ├── StockIndicator.tsx
│   │   │   │   └── StockIndicator.stories.tsx
│   │   │   ├── DeliveryEstimate/         # Icon + "Arrives by Thu, Apr 24"
│   │   │   │   ├── DeliveryEstimate.tsx
│   │   │   │   └── DeliveryEstimate.stories.tsx
│   │   │   ├── FreeShippingProgress/     # ProgressBar + "Add $12 more for free shipping"
│   │   │   │   ├── FreeShippingProgress.tsx
│   │   │   │   └── FreeShippingProgress.stories.tsx
│   │   │   ├── SplitPaymentBadge/        # "4 payments of $12 with Klarna"
│   │   │   │   ├── SplitPaymentBadge.tsx
│   │   │   │   └── SplitPaymentBadge.stories.tsx
│   │   │   ├── TrustSignalItem/          # Single trust row — Icon + Text
│   │   │   │   ├── TrustSignalItem.tsx
│   │   │   │   └── TrustSignalItem.stories.tsx
│   │   │   ├── SecureCheckoutBadge/      # Lock icon + "Secure Checkout"
│   │   │   │   ├── SecureCheckoutBadge.tsx
│   │   │   │   └── SecureCheckoutBadge.stories.tsx
│   │   │   ├── VerifiedBuyerTag/         # Review attribution badge
│   │   │   │   ├── VerifiedBuyerTag.tsx
│   │   │   │   └── VerifiedBuyerTag.stories.tsx
│   │   │   ├── PressQuote/               # Logo + "As seen in…"
│   │   │   │   ├── PressQuote.tsx
│   │   │   │   └── PressQuote.stories.tsx
│   │   │   ├── AnnouncementBar/          # Single promo strip
│   │   │   │   ├── AnnouncementBar.tsx
│   │   │   │   └── AnnouncementBar.stories.tsx
│   │   │   ├── CountdownUnit/            # CountdownTimer digit + label
│   │   │   │   ├── CountdownUnit.tsx
│   │   │   │   └── CountdownUnit.stories.tsx
│   │   │   ├── FormField/                # Label + Input + error/hint Text
│   │   │   │   ├── FormField.tsx
│   │   │   │   └── FormField.stories.tsx
│   │   │   ├── SearchBar/                # Input + magnifier IconButton + clear
│   │   │   │   ├── SearchBar.tsx
│   │   │   │   └── SearchBar.stories.tsx
│   │   │   ├── QuantitySelector/         # IconButton(−) + QuantityInput + IconButton(+)
│   │   │   │   ├── QuantitySelector.tsx
│   │   │   │   └── QuantitySelector.stories.tsx
│   │   │   ├── PriceRange/               # Min/max Inputs + apply Button
│   │   │   │   ├── PriceRange.tsx
│   │   │   │   └── PriceRange.stories.tsx
│   │   │   ├── CouponInput/              # Input + "Apply" Button
│   │   │   │   ├── CouponInput.tsx
│   │   │   │   └── CouponInput.stories.tsx
│   │   │   ├── PasswordInput/            # Input + toggle-visibility IconButton
│   │   │   │   ├── PasswordInput.tsx
│   │   │   │   └── PasswordInput.stories.tsx
│   │   │   ├── NewsletterInput/          # Email Input + Subscribe Button
│   │   │   │   ├── NewsletterInput.tsx
│   │   │   │   └── NewsletterInput.stories.tsx
│   │   │   ├── BackInStockForm/          # Input + "Notify me" Button
│   │   │   │   ├── BackInStockForm.tsx
│   │   │   │   └── BackInStockForm.stories.tsx
│   │   │   ├── ColorSwatch/              # Clickable colour circle with selected ring
│   │   │   │   ├── ColorSwatch.tsx
│   │   │   │   └── ColorSwatch.stories.tsx
│   │   │   ├── SwatchGroup/              # Row of ColorSwatches
│   │   │   │   ├── SwatchGroup.tsx
│   │   │   │   └── SwatchGroup.stories.tsx
│   │   │   ├── SizeOption/               # Single size button S/M/L/XL
│   │   │   │   ├── SizeOption.tsx
│   │   │   │   └── SizeOption.stories.tsx
│   │   │   ├── SizeSelector/             # Row of SizeOptions
│   │   │   │   ├── SizeSelector.tsx
│   │   │   │   └── SizeSelector.stories.tsx
│   │   │   ├── SizeGuideLink/            # Icon + Link — opens SizeGuideModal
│   │   │   │   ├── SizeGuideLink.tsx
│   │   │   │   └── SizeGuideLink.stories.tsx
│   │   │   ├── SubscribeSave/            # RadioButton pair + DiscountBadge
│   │   │   │   ├── SubscribeSave.tsx
│   │   │   │   └── SubscribeSave.stories.tsx
│   │   │   ├── BundleOfferItem/          # Checkbox + Image + PriceTag
│   │   │   │   ├── BundleOfferItem.tsx
│   │   │   │   └── BundleOfferItem.stories.tsx
│   │   │   ├── GiftWrapOption/           # Checkbox + Icon + Text
│   │   │   │   ├── GiftWrapOption.tsx
│   │   │   │   └── GiftWrapOption.stories.tsx
│   │   │   ├── CompareCheckbox/          # "Add to compare" Checkbox + Text
│   │   │   │   ├── CompareCheckbox.tsx
│   │   │   │   └── CompareCheckbox.stories.tsx
│   │   │   ├── ImageThumbnail/           # Image tile with optional overlay
│   │   │   │   ├── ImageThumbnail.tsx
│   │   │   │   └── ImageThumbnail.stories.tsx
│   │   │   ├── ProductVideo/             # VideoPlayButton + thumbnail Image
│   │   │   │   ├── ProductVideo.tsx
│   │   │   │   └── ProductVideo.stories.tsx
│   │   │   ├── ARViewerTrigger/          # Icon + "View in your space"
│   │   │   │   ├── ARViewerTrigger.tsx
│   │   │   │   └── ARViewerTrigger.stories.tsx
│   │   │   ├── SearchSuggestionItem/     # Icon/Image + Text for autocomplete row
│   │   │   │   ├── SearchSuggestionItem.tsx
│   │   │   │   └── SearchSuggestionItem.stories.tsx
│   │   │   ├── SearchCategoryHint/       # "in Shoes" grouping label
│   │   │   │   ├── SearchCategoryHint.tsx
│   │   │   │   └── SearchCategoryHint.stories.tsx
│   │   │   ├── RecentSearchItem/         # Clock icon + Text + remove IconButton
│   │   │   │   ├── RecentSearchItem.tsx
│   │   │   │   └── RecentSearchItem.stories.tsx
│   │   │   ├── NoResultsHint/            # Icon + Text + suggested link
│   │   │   │   ├── NoResultsHint.tsx
│   │   │   │   └── NoResultsHint.stories.tsx
│   │   │   ├── CartItemMeta/             # Image + name + variant Text
│   │   │   │   ├── CartItemMeta.tsx
│   │   │   │   └── CartItemMeta.stories.tsx
│   │   │   ├── CartItemPrice/            # QuantitySelector + PriceTag
│   │   │   │   ├── CartItemPrice.tsx
│   │   │   │   └── CartItemPrice.stories.tsx
│   │   │   ├── OrderSummaryLine/         # Label/value row — subtotal, tax, shipping
│   │   │   │   ├── OrderSummaryLine.tsx
│   │   │   │   └── OrderSummaryLine.stories.tsx
│   │   │   ├── ShippingOption/           # RadioButton + label + time + price
│   │   │   │   ├── ShippingOption.tsx
│   │   │   │   └── ShippingOption.stories.tsx
│   │   │   ├── PaymentMethodBadge/       # Visa / PayPal / Klarna icon + label
│   │   │   │   ├── PaymentMethodBadge.tsx
│   │   │   │   └── PaymentMethodBadge.stories.tsx
│   │   │   ├── StepIndicator/            # Numbered Dot + step label
│   │   │   │   ├── StepIndicator.tsx
│   │   │   │   └── StepIndicator.stories.tsx
│   │   │   ├── OrderStatusStep/          # Icon + label + timestamp
│   │   │   │   ├── OrderStatusStep.tsx
│   │   │   │   └── OrderStatusStep.stories.tsx
│   │   │   ├── RefundStatusBadge/        # Lozenge — Requested/Approved/Issued
│   │   │   │   ├── RefundStatusBadge.tsx
│   │   │   │   └── RefundStatusBadge.stories.tsx
│   │   │   ├── NavLink/                  # Icon? + Text with active state
│   │   │   │   ├── NavLink.tsx
│   │   │   │   └── NavLink.stories.tsx
│   │   │   ├── BreadcrumbItem/           # Link + separator Icon
│   │   │   │   ├── BreadcrumbItem.tsx
│   │   │   │   └── BreadcrumbItem.stories.tsx
│   │   │   ├── PaginationItem/           # Page number Button
│   │   │   │   ├── PaginationItem.tsx
│   │   │   │   └── PaginationItem.stories.tsx
│   │   │   ├── SortSelect/               # Label + Select "Sort by: Price ↑"
│   │   │   │   ├── SortSelect.tsx
│   │   │   │   └── SortSelect.stories.tsx
│   │   │   ├── FilterChip/               # Tag + remove IconButton
│   │   │   │   ├── FilterChip.tsx
│   │   │   │   └── FilterChip.stories.tsx
│   │   │   ├── TabItem/                  # Tab trigger Button with active/inactive state
│   │   │   │   ├── TabItem.tsx
│   │   │   │   └── TabItem.stories.tsx
│   │   │   ├── AccordionItem/            # Trigger Button + collapsible panel
│   │   │   │   ├── AccordionItem.tsx
│   │   │   │   └── AccordionItem.stories.tsx
│   │   │   ├── CarouselSlide/            # Single slide wrapper
│   │   │   │   ├── CarouselSlide.tsx
│   │   │   │   └── CarouselSlide.stories.tsx
│   │   │   ├── CarouselDot/              # Carousel position indicator
│   │   │   │   ├── CarouselDot.tsx
│   │   │   │   └── CarouselDot.stories.tsx
│   │   │   ├── CarouselArrow/            # Prev/next slide IconButton
│   │   │   │   ├── CarouselArrow.tsx
│   │   │   │   └── CarouselArrow.stories.tsx
│   │   │   ├── WishlistButton/           # Heart toggle — filled / outline
│   │   │   │   ├── WishlistButton.tsx
│   │   │   │   └── WishlistButton.stories.tsx
│   │   │   ├── ShareButton/              # IconButton + share label
│   │   │   │   ├── ShareButton.tsx
│   │   │   │   └── ShareButton.stories.tsx
│   │   │   ├── NotificationBadge/        # Cart Icon + item count Dot
│   │   │   │   ├── NotificationBadge.tsx
│   │   │   │   └── NotificationBadge.stories.tsx
│   │   │   ├── SocialLink/               # Icon + VisuallyHidden label
│   │   │   │   ├── SocialLink.tsx
│   │   │   │   └── SocialLink.stories.tsx
│   │   │   ├── ReviewCard/               # Avatar + Text + RatingSummary + VerifiedBuyerTag
│   │   │   │   ├── ReviewCard.tsx
│   │   │   │   └── ReviewCard.stories.tsx
│   │   │   ├── CategoryTile/             # Image + Text label
│   │   │   │   ├── CategoryTile.tsx
│   │   │   │   └── CategoryTile.stories.tsx
│   │   │   ├── AddressLine/              # Icon + formatted address Text
│   │   │   │   ├── AddressLine.tsx
│   │   │   │   └── AddressLine.stories.tsx
│   │   │   └── TrustBadge/               # Icon + Heading + Text trust block
│   │   │       ├── TrustBadge.tsx
│   │   │       └── TrustBadge.stories.tsx
│   │   ├── organisms/
│   │   │   ├── HeroBanner/               # Full-width image/video + Heading + Button(s)
│   │   │   │   ├── HeroBanner.tsx
│   │   │   │   └── HeroBanner.stories.tsx
│   │   │   ├── PromoBanner/              # AnnouncementBar + optional CountdownUnit
│   │   │   │   ├── PromoBanner.tsx
│   │   │   │   └── PromoBanner.stories.tsx
│   │   │   ├── CategoryBanner/           # Image + CategoryTile overlay
│   │   │   │   ├── CategoryBanner.tsx
│   │   │   │   └── CategoryBanner.stories.tsx
│   │   │   ├── FreeShippingBanner/       # Site-wide FreeShippingProgress
│   │   │   │   ├── FreeShippingBanner.tsx
│   │   │   │   └── FreeShippingBanner.stories.tsx
│   │   │   ├── HeroCarousel/             # Auto-play slides + dots + arrows
│   │   │   │   ├── HeroCarousel.tsx
│   │   │   │   └── HeroCarousel.stories.tsx
│   │   │   ├── ProductCarousel/          # Horizontal scroll of ProductCards + arrows
│   │   │   │   ├── ProductCarousel.tsx
│   │   │   │   └── ProductCarousel.stories.tsx
│   │   │   ├── BannerCarousel/           # Rotating HeroBanners
│   │   │   │   ├── BannerCarousel.tsx
│   │   │   │   └── BannerCarousel.stories.tsx
│   │   │   ├── ThumbnailGallery/         # Main Image + ImageThumbnail strip + zoom
│   │   │   │   ├── ThumbnailGallery.tsx
│   │   │   │   └── ThumbnailGallery.stories.tsx
│   │   │   ├── ProductVideoGallery/      # Mixed image + video ThumbnailGallery
│   │   │   │   ├── ProductVideoGallery.tsx
│   │   │   │   └── ProductVideoGallery.stories.tsx
│   │   │   ├── Accordion/                # Stacked AccordionItems — FAQ, filters
│   │   │   │   ├── Accordion.tsx
│   │   │   │   └── Accordion.stories.tsx
│   │   │   ├── TabGroup/                 # Horizontal or vertical TabItems + panels
│   │   │   │   ├── TabGroup.tsx
│   │   │   │   └── TabGroup.stories.tsx
│   │   │   ├── Drawer/                   # Slide-in panel — cart, filters, mobile nav
│   │   │   │   ├── Drawer.tsx
│   │   │   │   └── Drawer.stories.tsx
│   │   │   ├── Modal/                    # Overlay + dialog — quick view, confirm, zoom
│   │   │   │   ├── Modal.tsx
│   │   │   │   └── Modal.stories.tsx
│   │   │   ├── Navbar/                   # Logo + NavLinks + SearchBar + cart + Avatar
│   │   │   │   ├── Navbar.tsx
│   │   │   │   └── Navbar.stories.tsx
│   │   │   ├── MegaMenu/                 # Multi-column dropdown from Navbar item
│   │   │   │   ├── MegaMenu.tsx
│   │   │   │   └── MegaMenu.stories.tsx
│   │   │   ├── Breadcrumbs/              # Ordered BreadcrumbItems
│   │   │   │   ├── Breadcrumbs.tsx
│   │   │   │   └── Breadcrumbs.stories.tsx
│   │   │   ├── Pagination/               # Prev/next + PaginationItems
│   │   │   │   ├── Pagination.tsx
│   │   │   │   └── Pagination.stories.tsx
│   │   │   ├── FilterSidebar/            # Stacked Accordions + FilterChips
│   │   │   │   ├── FilterSidebar.tsx
│   │   │   │   └── FilterSidebar.stories.tsx
│   │   │   ├── SortFilterBar/            # SortSelect + FilterChips + result count
│   │   │   │   ├── SortFilterBar.tsx
│   │   │   │   └── SortFilterBar.stories.tsx
│   │   │   ├── Header/                   # Navbar + optional PromoBanner
│   │   │   │   ├── Header.tsx
│   │   │   │   └── Header.stories.tsx
│   │   │   ├── Footer/                   # Logo + SocialLinks + columns + NewsletterInput
│   │   │   │   ├── Footer.tsx
│   │   │   │   └── Footer.stories.tsx
│   │   │   ├── SearchOverlay/            # Full-screen search: bar + recents + suggestions
│   │   │   │   ├── SearchOverlay.tsx
│   │   │   │   └── SearchOverlay.stories.tsx
│   │   │   ├── SearchResultsHeader/      # Query + result count + SortFilterBar
│   │   │   │   ├── SearchResultsHeader.tsx
│   │   │   │   └── SearchResultsHeader.stories.tsx
│   │   │   ├── EmptySearchResults/       # EmptyState + popular categories + SearchBar
│   │   │   │   ├── EmptySearchResults.tsx
│   │   │   │   └── EmptySearchResults.stories.tsx
│   │   │   ├── ProductCard/              # ImageThumbnail + badges + PriceDisplay + Button
│   │   │   │   ├── ProductCard.tsx
│   │   │   │   └── ProductCard.stories.tsx
│   │   │   ├── ProductGrid/              # Responsive grid of ProductCards
│   │   │   │   ├── ProductGrid.tsx
│   │   │   │   └── ProductGrid.stories.tsx
│   │   │   ├── ProductImageGallery/      # ThumbnailGallery with zoom
│   │   │   │   ├── ProductImageGallery.tsx
│   │   │   │   └── ProductImageGallery.stories.tsx
│   │   │   ├── ProductDetails/           # Accordion — description/specs/size guide
│   │   │   │   ├── ProductDetails.tsx
│   │   │   │   └── ProductDetails.stories.tsx
│   │   │   ├── QuickViewModal/           # Modal: gallery + price + size + add-to-cart
│   │   │   │   ├── QuickViewModal.tsx
│   │   │   │   └── QuickViewModal.stories.tsx
│   │   │   ├── StickyAddToCart/          # Fixed bar after scrolling past buy box
│   │   │   │   ├── StickyAddToCart.tsx
│   │   │   │   └── StickyAddToCart.stories.tsx
│   │   │   ├── SizeGuideModal/           # Modal with measurement table + fit tips
│   │   │   │   ├── SizeGuideModal.tsx
│   │   │   │   └── SizeGuideModal.stories.tsx
│   │   │   ├── FrequentlyBoughtTogether/ # BundleOfferItems + combined price + add-all
│   │   │   │   ├── FrequentlyBoughtTogether.tsx
│   │   │   │   └── FrequentlyBoughtTogether.stories.tsx
│   │   │   ├── ARViewerPanel/            # AR viewer canvas + instructions
│   │   │   │   ├── ARViewerPanel.tsx
│   │   │   │   └── ARViewerPanel.stories.tsx
│   │   │   ├── ReviewList/               # RatingSummary + ReviewCards + Pagination
│   │   │   │   ├── ReviewList.tsx
│   │   │   │   └── ReviewList.stories.tsx
│   │   │   ├── RelatedProducts/          # ProductCarousel with heading
│   │   │   │   ├── RelatedProducts.tsx
│   │   │   │   └── RelatedProducts.stories.tsx
│   │   │   ├── RecentlyViewedProducts/   # Horizontal scroll ProductCards
│   │   │   │   ├── RecentlyViewedProducts.tsx
│   │   │   │   └── RecentlyViewedProducts.stories.tsx
│   │   │   ├── CartDrawer/               # Drawer: cart items + totals
│   │   │   │   ├── CartDrawer.tsx
│   │   │   │   └── CartDrawer.stories.tsx
│   │   │   ├── CartSummary/              # Full-page order summary + CouponInput
│   │   │   │   ├── CartSummary.tsx
│   │   │   │   └── CartSummary.stories.tsx
│   │   │   ├── CartCrossSell/            # "You may also need" inside CartDrawer
│   │   │   │   ├── CartCrossSell.tsx
│   │   │   │   └── CartCrossSell.stories.tsx
│   │   │   ├── GuestCheckoutPrompt/      # Guest / sign-in / sign-up choice
│   │   │   │   ├── GuestCheckoutPrompt.tsx
│   │   │   │   └── GuestCheckoutPrompt.stories.tsx
│   │   │   ├── CheckoutStepper/          # StepIndicator row for checkout flow
│   │   │   │   ├── CheckoutStepper.tsx
│   │   │   │   └── CheckoutStepper.stories.tsx
│   │   │   ├── AddressAutocomplete/      # FormField with address suggestions
│   │   │   │   ├── AddressAutocomplete.tsx
│   │   │   │   └── AddressAutocomplete.stories.tsx
│   │   │   ├── OrderSummaryCollapsible/  # Accordion CartSummary for mobile checkout
│   │   │   │   ├── OrderSummaryCollapsible.tsx
│   │   │   │   └── OrderSummaryCollapsible.stories.tsx
│   │   │   ├── PromoCodePanel/           # CouponInput + applied discount line
│   │   │   │   ├── PromoCodePanel.tsx
│   │   │   │   └── PromoCodePanel.stories.tsx
│   │   │   ├── GiftOptionsPanel/         # GiftWrapOption + gift message Textarea
│   │   │   │   ├── GiftOptionsPanel.tsx
│   │   │   │   └── GiftOptionsPanel.stories.tsx
│   │   │   ├── ShippingForm/             # FormFields + ShippingOption list
│   │   │   │   ├── ShippingForm.tsx
│   │   │   │   └── ShippingForm.stories.tsx
│   │   │   ├── PaymentForm/              # FormFields + PaymentMethodBadge selector
│   │   │   │   ├── PaymentForm.tsx
│   │   │   │   └── PaymentForm.stories.tsx
│   │   │   ├── OrderConfirmation/        # Icon + Heading + order summary
│   │   │   │   ├── OrderConfirmation.tsx
│   │   │   │   └── OrderConfirmation.stories.tsx
│   │   │   ├── OrderTracker/             # Timeline of OrderStatusSteps
│   │   │   │   ├── OrderTracker.tsx
│   │   │   │   └── OrderTracker.stories.tsx
│   │   │   ├── OrderRow/                 # Lozenge + order number + date + price + Button
│   │   │   │   ├── OrderRow.tsx
│   │   │   │   └── OrderRow.stories.tsx
│   │   │   ├── OrderTable/               # Table of OrderRows + Pagination
│   │   │   │   ├── OrderTable.tsx
│   │   │   │   └── OrderTable.stories.tsx
│   │   │   ├── ReturnRequestForm/        # FormFields + reason Select + Checkboxes
│   │   │   │   ├── ReturnRequestForm.tsx
│   │   │   │   └── ReturnRequestForm.stories.tsx
│   │   │   ├── ReorderButton/            # One-tap reorder for a past order
│   │   │   │   ├── ReorderButton.tsx
│   │   │   │   └── ReorderButton.stories.tsx
│   │   │   ├── ProfileForm/              # FormFields + Avatar upload + save Button
│   │   │   │   ├── ProfileForm.tsx
│   │   │   │   └── ProfileForm.stories.tsx
│   │   │   ├── AddressCard/              # AddressLine + edit/delete IconButtons
│   │   │   │   ├── AddressCard.tsx
│   │   │   │   └── AddressCard.stories.tsx
│   │   │   ├── ToastStack/               # Queued Toast notifications — fixed position
│   │   │   │   ├── ToastStack.tsx
│   │   │   │   └── ToastStack.stories.tsx
│   │   │   ├── EmptyState/               # Icon + Heading + Text + optional Button
│   │   │   │   ├── EmptyState.tsx
│   │   │   │   └── EmptyState.stories.tsx
│   │   │   ├── SkipNavigation/           # SkipLink as first element in body — WCAG 2.4.1
│   │   │   │   ├── SkipNavigation.tsx
│   │   │   │   └── SkipNavigation.stories.tsx
│   │   │   ├── CookieBanner/             # Text + accept/decline Buttons — fixed bottom
│   │   │   │   ├── CookieBanner.tsx
│   │   │   │   └── CookieBanner.stories.tsx
│   │   │   ├── CookieConsentManager/     # Granular Accordion with category toggles
│   │   │   │   ├── CookieConsentManager.tsx
│   │   │   │   └── CookieConsentManager.stories.tsx
│   │   │   ├── AgeVerificationGate/      # Blocking Modal for restricted pages
│   │   │   │   ├── AgeVerificationGate.tsx
│   │   │   │   └── AgeVerificationGate.stories.tsx
│   │   │   ├── LanguageCurrencySwitcher/ # Select pair for locale in Header/Footer
│   │   │   │   ├── LanguageCurrencySwitcher.tsx
│   │   │   │   └── LanguageCurrencySwitcher.stories.tsx
│   │   │   ├── LiveChatWidget/           # Floating IconButton + chat Drawer
│   │   │   │   ├── LiveChatWidget.tsx
│   │   │   │   └── LiveChatWidget.stories.tsx
│   │   │   └── TooltipPortal/            # Tooltip in portal — escapes overflow:hidden
│   │   │       ├── TooltipPortal.tsx
│   │   │       └── TooltipPortal.stories.tsx
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
