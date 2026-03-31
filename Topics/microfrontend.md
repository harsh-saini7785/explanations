# Microfrontend Architecture — Complete Guide

> A comprehensive reference covering what microfrontends are, how to implement them with Module Federation and Vite, and how to solve every major challenge with real code examples.

---

## Table of Contents

1. [What is a Microfrontend?](#what-is-a-microfrontend)
2. [How Module Federation Works](#how-module-federation-works)
3. [Setting Up Vite + Module Federation](#setting-up-vite--module-federation)
4. [Challenge 1 — Shared Dependency Version Conflicts](#challenge-1--shared-dependency-version-conflicts)
5. [Challenge 2 — Build-time vs Runtime Federation (Vite Limitation)](#challenge-2--build-time-vs-runtime-federation-vite-limitation)
6. [Challenge 3 — CSS Isolation and Style Leakage](#challenge-3--css-isolation-and-style-leakage)
7. [Challenge 4 — Cross-MFE State Management](#challenge-4--cross-mfe-state-management)
8. [Challenge 5 — TypeScript Types Across Remotes](#challenge-5--typescript-types-across-remotes)
9. [Challenge 6 — Initial Load Performance](#challenge-6--initial-load-performance)
10. [Challenge 7 — Routing Ownership Conflicts](#challenge-7--routing-ownership-conflicts)
11. [Challenge 8 — Local Development Experience and HMR](#challenge-8--local-development-experience-and-hmr)
12. [Dev vs Production — What Changes](#dev-vs-production--what-changes)
13. [Mock Remotes Pattern](#mock-remotes-pattern)
14. [When NOT to Use Microfrontends](#when-not-to-use-microfrontends)

---

## What is a Microfrontend?

A **microfrontend** is a architectural pattern where a large frontend application is broken into smaller, independently deployable pieces — each owned by a separate team, built with its own tech stack, and composed together at runtime in the browser.

It is the frontend equivalent of microservices on the backend.

```
┌──────────────────────────────────────────────────┐
│                  Shell / Host App                │
│         (composes all MFEs at runtime)           │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Nav MFE     │  │Checkout MFE │  │Profile   │ │
│  │ Team A      │  │Team B·React │  │MFE Team C│ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  Shared Layer — React, ReactDOM, tokens  │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

### Key benefits

- **Independent deployability** — Team B can ship checkout on Tuesday without coordinating with Team A
- **Tech autonomy** — One team can use React 18, another Vue 3
- **Isolated failures** — A crash in the profile MFE doesn't take down the whole app
- **Parallel development** — Teams work without blocking each other

### Key trade-offs

- Significantly more complex tooling and configuration
- Network overhead from loading multiple bundles
- Requires strong team contracts (props, events, types)
- Local development experience is harder

---

## How Module Federation Works

Module Federation (originally a Webpack 5 feature) lets apps **expose** and **consume** JavaScript modules across separately deployed bundles — at runtime in the browser, without any npm install between teams.

Each app plays one of two roles:

- **Remote** — exposes components/modules that others can use
- **Host (Shell)** — consumes modules from one or more remotes

When the shell loads in the browser, it fetches a small `remoteEntry.js` file from each remote. This manifest lists what the remote exposes and which shared dependencies it has. The browser then loads only the chunks it needs.

```
Browser loads shell
    │
    ├── fetches checkout/remoteEntry.js  →  learns what checkout exposes
    ├── fetches products/remoteEntry.js  →  learns what products exposes
    │
    └── user navigates to /checkout
            │
            └── browser fetches checkout/Cart chunk  →  renders Cart
```

---

## Setting Up Vite + Module Federation

Install the plugin in every MFE and the shell:

```bash
npm install @originjs/vite-plugin-federation --save-dev
```

### Remote app (e.g. checkout-mfe)

```ts
// checkout-mfe/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'checkout',
      filename: 'remoteEntry.js',
      exposes: {
        './Cart':     './src/components/Cart.tsx',
        './Checkout': './src/pages/Checkout.tsx',
      },
      shared: ['react', 'react-dom'],
    }),
  ],
  build: {
    target: 'esnext',  // required for Module Federation
  },
})
```

### Shell / Host app

```ts
// shell/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'host',
      remotes: {
        checkout: 'http://localhost:5001/assets/remoteEntry.js',
        products: 'http://localhost:5002/assets/remoteEntry.js',
      },
      shared: ['react', 'react-dom'],
    }),
  ],
  build: {
    target: 'esnext',
  },
})
```

### Consuming a remote in the shell

```tsx
// shell/src/pages/ShopPage.tsx
import React from 'react'

// This import is resolved at runtime — not from node_modules
const Cart = React.lazy(() => import('checkout/Cart'))

export default function ShopPage() {
  return (
    <React.Suspense fallback={<div>Loading cart...</div>}>
      <Cart userId="u123" onCheckout={() => console.log('checkout!')} />
    </React.Suspense>
  )
}
```

---

## Challenge 1 — Shared Dependency Version Conflicts

### The problem

If two MFEs load different versions of React, the browser ends up with **two separate React objects in memory**. React uses a module-level singleton called `ReactCurrentDispatcher` to track which component is currently rendering. When there are two React instances, `react-dom` sets the dispatcher on Instance A, but a component in Instance B reads from its own dispatcher — which is still `null`. Every hook call throws:

```
Error: Invalid hook call. Hooks can only be called inside of the body
of a function component. You might have more than one copy of React.
```

This affects every hook — `useState`, `useEffect`, `useContext`, `useRef`, `useMemo` — because they all go through the same `resolveDispatcher()` call internally.

### Why it happens

```ts
// Inside react/src/ReactHooks.js
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current
  // react-dom set this on Instance A's object
  // but this code belongs to Instance B — its .current is still null
  if (dispatcher === null) {
    throw new Error('Invalid hook call...')
  }
  return dispatcher
}

export function useState(init) {
  return resolveDispatcher().useState(init)  // crashes here
}
```

### The solution — `singleton: true`

Apply this to **every MFE and the shell** — the config must be identical everywhere:

```ts
// vite.config.ts (apply to ALL apps)
federation({
  shared: {
    react: {
      singleton: true,           // only ONE React instance allowed in the page
      requiredVersion: '^18.2.0', // warns if a remote tries to use a different version
      eager: true,               // load upfront, not lazily
    },
    'react-dom': {
      singleton: true,
      requiredVersion: '^18.2.0',
      eager: true,
    },
    'react-router-dom': {       // routers must also be singletons
      singleton: true,
      requiredVersion: '^6.0.0',
    },
  },
})
```

With `singleton: true`, the Module Federation runtime intercepts every `import 'react'` across all bundles and returns the **same JavaScript object** — same memory address — to everyone. The dispatcher is set once and visible everywhere.

> **Pro tip:** Pin all teams to exact versions in a monorepo root `package.json`. This is the source of truth so all MFEs resolve to the same `node_modules`.

---

## Challenge 2 — Build-time vs Runtime Federation (Vite Limitation)

### The problem

Webpack 5's Module Federation was designed for **runtime** operation — bundles exchange modules dynamically while the browser runs. Vite works differently: in dev mode it serves raw, unbundled ES modules directly from disk. There is no bundle, so there is no `remoteEntry.js`.

```
Vite dev mode:  serves raw ESM files  →  no remoteEntry.js exists
Module Federation needs:  remoteEntry.js  →  404 error, blank page
```

### What this forces you to do in dev

Every remote must be **built** (not just served) before the shell can consume it:

```bash
# For each remote — two processes each
cd checkout-mfe
vite build --watch        # continuously rebuilds into dist/
vite preview --port 5001  # serves the dist/ folder

# Shell — this one gets normal HMR
cd shell
vite dev --port 3000
```

### The solutions

**Option 1: `vite build --watch` + `vite preview` per remote (standard approach)**

```json
// package.json in each remote
{
  "scripts": {
    "dev":     "vite build --watch",
    "preview": "vite preview --port 5001"
  }
}
```

**Option 2: Switch to `@module-federation/vite` (better DX)**

Built by the original Module Federation team. Uses Vite's virtual module system to approximate runtime federation without requiring a full build in dev mode:

```bash
npm install @module-federation/vite
```

```ts
// vite.config.ts
import { federation } from '@module-federation/vite'

export default defineConfig({
  plugins: [
    federation({
      name: 'shell',
      remotes: { checkout: 'checkout@http://localhost:5001/mf-manifest.json' },
      shared: { react: { singleton: true } },
    }),
  ],
})
```

**Option 3: Mock remotes (best daily DX — see [Mock Remotes Pattern](#mock-remotes-pattern))**

Use local fake components during development. Zero extra processes, full HMR.

---

## Challenge 3 — CSS Isolation and Style Leakage

### The problem

Each MFE ships its own CSS. Without isolation, class names like `.btn`, `.card`, or `.container` collide globally — whichever stylesheet loads last wins, causing visual bugs that only appear when two MFEs are on the same page simultaneously.

Tailwind is especially risky — both MFEs generate identical class names (`.p-4`, `.flex`, `.btn`) but with different configuration values.

```css
/* checkout-mfe — loads first */
.btn { background: blue; padding: 8px 16px; }

/* shell — loads second, overwrites checkout's .btn */
.btn { background: green; padding: 4px 8px; }

/* Result: checkout's buttons are now green — a bug */
```

### Solution 1 — CSS Modules (recommended)

Vite supports CSS Modules out of the box. Class names are automatically scoped to the component:

```css
/* Button.module.css */
.btn {
  background: blue;
  padding: 8px 16px;
}
```

```tsx
// Button.tsx
import styles from './Button.module.css'

// Vite compiles .btn → .checkout__btn__x7k2f (unique hash)
export const Button = () => (
  <button className={styles.btn}>Click me</button>
)
```

No global collision possible — every class name is unique across the entire page.

### Solution 2 — Tailwind prefix per MFE

```js
// checkout-mfe/tailwind.config.js
module.exports = {
  prefix: 'co-',  // all classes become co-flex, co-p-4, co-btn
}

// shell/tailwind.config.js
module.exports = {
  prefix: 'sh-',  // all classes become sh-flex, sh-p-4, sh-btn
}
```

### Solution 3 — Shadow DOM (strongest isolation)

Full CSS encapsulation — nothing leaks in or out:

```tsx
import ReactDOM from 'react-dom/client'

export function mountCheckout(element: HTMLElement) {
  const shadow = element.attachShadow({ mode: 'open' })
  const mountPoint = document.createElement('div')
  shadow.appendChild(mountPoint)
  ReactDOM.createRoot(mountPoint).render(<CheckoutApp />)
}
```

> **Caveat:** Shadow DOM breaks React portals (tooltips, modals) and third-party component libraries like MUI that portal to `document.body`. Use CSS Modules unless you need absolute isolation.

---

## Challenge 4 — Cross-MFE State Management

### The problem

MFEs are isolated JS bundles. You cannot import a Zustand store from one MFE into another — they live in separate JS contexts. But real apps need shared state: auth tokens, cart counts, user locale, theme.

```ts
// WRONG — direct store import across MFE bundles
// In cart-mfe/src/Cart.tsx:
import { useAuthStore } from 'auth-mfe/store'
// Module not found — cross-bundle imports fail at runtime
```

### Solution 1 — Custom events (loose coupling, recommended)

Native browser custom events work across all JS contexts on the page:

```ts
// auth-mfe: emit after login
window.dispatchEvent(new CustomEvent('auth:login', {
  detail: { userId: 'u1', token: 'xyz', name: 'Amit' },
}))

// cart-mfe: listen anywhere
window.addEventListener('auth:login', (e: CustomEvent) => {
  const { userId, token } = e.detail
  setUser({ userId, token })
})

// Define event types in a shared package for type safety
// packages/shared-events/src/index.ts
export const AUTH_LOGIN = 'auth:login'
export const CART_UPDATED = 'cart:updated'
```

### Solution 2 — Shell-owned context (tight coupling)

The shell owns the state and passes it as props to every remote:

```tsx
// shell/src/App.tsx
const [user, setUser] = useState<User | null>(null)

return (
  <Routes>
    <Route path="/checkout/*" element={
      <Suspense fallback={<Spinner />}>
        <RemoteCheckout
          user={user}               // shell passes state down
          onAuthRequired={() => navigate('/login')}
        />
      </Suspense>
    } />
  </Routes>
)
```

### Solution 3 — Shell exposes a shared store module

```ts
// shell/vite.config.ts — expose the store as a module
federation({
  exposes: {
    './store': './src/store/sharedStore.ts',
  },
  shared: { react: { singleton: true } },
})

// any remote can then import it (it's the same singleton object)
const { useUser } = await import('shell/store')
```

> **Rule of thumb:** Use custom events for truly independent MFEs (best for scaling). Use shell props when teams are tightly coordinated. Avoid option 3 unless you have strict version contracts — it creates hidden coupling between the shell and every remote.

---

## Challenge 5 — TypeScript Types Across Remotes

### The problem

Module Federation operates at runtime — TypeScript has no knowledge of remote modules at compile time. Every import from a remote shows a red underline, autocomplete breaks, and there is zero type safety at MFE boundaries.

```ts
// Host tries to use Cart from checkout remote
const Cart = React.lazy(() => import('checkout/Cart'))
//                                   ^^^^^^^^^^^^^^
// TS Error: Cannot find module 'checkout/Cart'
// Or: module resolves to 'any' — no type checking at all

<Cart userId={42} />   // no error — but userId should be string!
<Cart />               // no error — but userId is required!
```

### Solution 1 — Manual declaration files

```ts
// shell/src/types/remotes.d.ts
declare module 'checkout/Cart' {
  import { FC } from 'react'
  interface CartProps {
    userId: string
    onCheckout: () => void
  }
  const Cart: FC<CartProps>
  export default Cart
}

declare module 'checkout/Checkout' {
  import { FC } from 'react'
  const Checkout: FC<{ onSuccess: () => void }>
  export default Checkout
}
```

### Solution 2 — Shared types package in a monorepo (recommended)

```
packages/
  mfe-types/
    src/
      checkout.ts    ← CartProps, CheckoutProps
      products.ts    ← ListingProps, ProductProps
      events.ts      ← CustomEvent payloads
```

```ts
// packages/mfe-types/src/checkout.ts
export interface CartProps {
  userId: string
  onCheckout: () => void
}

export interface CheckoutProps {
  onSuccess: () => void
  onCancel: () => void
}
```

```ts
// checkout-mfe/src/components/Cart.tsx
import type { CartProps } from 'mfe-types/checkout'

export default function Cart({ userId, onCheckout }: CartProps) {
  // ...
}
```

```ts
// shell/src/types/remotes.d.ts
import type { CartProps } from 'mfe-types/checkout'

declare module 'checkout/Cart' {
  import { FC } from 'react'
  const Cart: FC<CartProps>
  export default Cart
}
```

When `CartProps` changes in the shared package, TypeScript errors appear immediately in all consumers — no runtime surprises.

### Solution 3 — Auto-generate types with `vite-plugin-dts`

```ts
// checkout-mfe/vite.config.ts
import dts from 'vite-plugin-dts'

export default defineConfig({
  plugins: [
    dts({ outDir: 'dist/types', include: ['src/components'] }),
    federation({ /* ... */ }),
  ],
})
```

Then publish `dist/types` to your internal npm registry or GitHub Packages.

---

## Challenge 6 — Initial Load Performance

### The problem

Each remote must fetch its `remoteEntry.js` manifest before the browser knows what JS chunks to load. With 5 MFEs, you get 5 sequential network requests before content appears — a visible waterfall that degrades Time to Interactive on slow connections.

```
t=0ms    shell.js loads and parses
t=400ms  nav remoteEntry.js fetch starts
t=800ms  nav chunks start loading
t=1200ms checkout remoteEntry.js (sequential after nav!)
t=1600ms checkout chunks
t=2000ms products remoteEntry.js
...
t=3500ms page finally renders  ← user sees blank screen this whole time
```

### Solution 1 — Preload critical remote entries

```html
<!-- shell/index.html -->
<!-- Fetch these in parallel with the shell bundle — no waiting -->
<link rel="modulepreload" href="https://cdn.example.com/nav/remoteEntry.js">
<link rel="modulepreload" href="https://cdn.example.com/checkout/remoteEntry.js">
```

### Solution 2 — Lazy-load non-critical MFEs by route

```tsx
// shell/src/App.tsx
// Only load checkout bundle when the user actually visits /checkout
const CheckoutPage = React.lazy(() => import('checkout/Checkout'))
const ProductsPage = React.lazy(() => import('products/Listing'))

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route
        path="/checkout/*"
        element={
          <Suspense fallback={<PageSkeleton />}>
            <CheckoutPage />
          </Suspense>
        }
      />
      <Route
        path="/products/*"
        element={
          <Suspense fallback={<PageSkeleton />}>
            <ProductsPage />
          </Suspense>
        }
      />
    </Routes>
  )
}
```

### Solution 3 — Correct cache headers

```
# remoteEntry.js — must always be fresh (changes on every deploy)
Cache-Control: no-cache

# chunk files — can be cached forever (content-hashed filenames)
Cache-Control: public, max-age=31536000, immutable
# e.g. checkout.chunk-a1b2c3d4.js — filename changes when content changes
```

### Solution 4 — `eager: true` for shared dependencies

```ts
federation({
  shared: {
    react:     { singleton: true, eager: true },  // load before any MFE
    'react-dom': { singleton: true, eager: true },
  },
})
```

---

## Challenge 7 — Routing Ownership Conflicts

### The problem

Both the shell and MFEs want to control routing. When a remote renders its own `<BrowserRouter>`, you get two routers fighting over the URL — navigation breaks, the back button stops working, and deep links fail.

```tsx
// WRONG — MFE creates its own top-level router
export default function CheckoutApp() {
  return (
    <BrowserRouter>  {/* creates a SECOND router — conflicts with shell */}
      <Routes>
        <Route path="/cart"    element={<Cart />} />
        <Route path="/payment" element={<Payment />} />
      </Routes>
    </BrowserRouter>
  )
}

// Shell navigates to /checkout/cart
// Shell router matches /checkout/* and renders CheckoutApp ✓
// Checkout's BrowserRouter sees full path /checkout/cart — no match ✗
```

### The solution — shell owns the router, MFEs use `<Routes>` only

```tsx
// shell/src/App.tsx — ONE BrowserRouter for the entire app
import { BrowserRouter, Routes, Route } from 'react-router-dom'

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/"           element={<Home />} />
        <Route path="/checkout/*" element={<RemoteCheckout />} />
        <Route path="/products/*" element={<RemoteProducts />} />
      </Routes>
    </BrowserRouter>
  )
}
```

```tsx
// checkout-mfe/src/CheckoutApp.tsx — no BrowserRouter, only Routes
import { Routes, Route } from 'react-router-dom'

export default function CheckoutApp() {
  return (
    // Inherits the shell's router context automatically
    <Routes>
      <Route path="cart"    element={<Cart />} />    {/* matches /checkout/cart */}
      <Route path="payment" element={<Payment />} /> {/* matches /checkout/payment */}
    </Routes>
  )
}
```

Also add `react-router-dom` to your singleton shared config:

```ts
shared: {
  'react-router-dom': { singleton: true, requiredVersion: '^6.0.0' },
}
```

> **Golden rule:** One `<BrowserRouter>` in the entire application, always owned by the shell. All MFEs use `<Routes>` only — they inherit the shell's router context automatically.

---

## Challenge 8 — Local Development Experience and HMR

### The problem

In a real project you might have 6 MFEs. Running all 6 locally just to work on one feature requires many terminal processes, uses significant RAM, and HMR does not cross federation boundaries — changes in a remote don't hot-reload the shell.

```
# To develop ONE feature that touches checkout:

Terminal 1: cd nav-mfe       && vite build --watch
Terminal 2:                     vite preview --port 5001
Terminal 3: cd checkout-mfe  && vite build --watch
Terminal 4:                     vite preview --port 5002
Terminal 5: cd products-mfe  && vite build --watch
Terminal 6:                     vite preview --port 5003
Terminal 7: cd shell         && vite dev

# Change Cart.tsx → rebuild takes 800ms → manual browser refresh
# RAM usage: ~1.2GB
```

### Solution 1 — Mock remotes (recommended for daily development)

Replace remote imports with local mock components during development. See the full [Mock Remotes Pattern](#mock-remotes-pattern) section below.

### Solution 2 — Turborepo for parallel startup

```json
// turbo.json
{
  "pipeline": {
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

```bash
# Starts all MFEs in parallel with one command
turbo run dev
```

### Solution 3 — Docker Compose

```yaml
# docker-compose.yml
services:
  checkout:
    build: ./checkout-mfe
    ports: ["5001:5001"]
    volumes: ["./checkout-mfe/src:/app/src"]  # live file watching

  products:
    build: ./products-mfe
    ports: ["5002:5002"]
    volumes: ["./products-mfe/src:/app/src"]

  shell:
    build: ./shell
    ports: ["3000:3000"]
    depends_on: [checkout, products]
```

```bash
docker-compose up  # all services start together
```

---

## Dev vs Production — What Changes

| What | Development | Production |
|---|---|---|
| Remote source | `localhost:500X` preview servers | CDN URLs (e.g. `cdn.example.com`) |
| Remote state | Must be built + previewed locally | Static files deployed independently |
| HMR | Shell only (not across boundaries) | N/A — users get latest on page load |
| `remoteEntry.js` | Served by `vite preview` | Uploaded to CDN by CI/CD |
| Processes needed | 2 per remote + 1 for shell | None — all static |
| Cache headers | N/A | `no-cache` for manifest, `immutable` for chunks |

```ts
// vite.config.ts — the only thing that changes between environments
federation({
  remotes: {
    checkout:
      process.env.NODE_ENV === 'production'
        ? 'https://cdn.example.com/checkout/remoteEntry.js'  // prod CDN
        : 'http://localhost:5001/assets/remoteEntry.js',     // local preview
  },
})
```

---

## Mock Remotes Pattern

Mocks are the most important DX improvement in Vite MF development. The idea: tell Vite to intercept remote imports and redirect them to local fake components — no network, no remote server, full HMR.

### How it works

Vite checks `resolve.alias` **before** doing anything else — before hitting the network, before asking the federation plugin. When it finds a match, it loads the local file directly.

```
Normal:  import('checkout/Cart')  →  fetch remoteEntry.js  →  fetch Cart chunk
Mocked:  import('checkout/Cart')  →  alias match  →  load ./src/mocks/checkout/Cart.tsx
```

### Step 1 — Update `vite.config.ts` in the shell

```ts
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

const useMocks = process.env.USE_MOCKS === 'true'

export default defineConfig({
  plugins: [
    federation({
      remotes: {
        checkout: 'http://localhost:5001/assets/remoteEntry.js',
        products: 'http://localhost:5002/assets/remoteEntry.js',
      },
    }),
  ],
  resolve: {
    alias: useMocks
      ? {
          // when USE_MOCKS=true, these intercept the remote imports
          'checkout/Cart':      './src/mocks/checkout/Cart.tsx',
          'checkout/Checkout':  './src/mocks/checkout/Checkout.tsx',
          'products/Listing':   './src/mocks/products/Listing.tsx',
        }
      : {}, // empty = use real remotes
  },
})
```

### Step 2 — Write mock components

```tsx
// shell/src/mocks/checkout/Cart.tsx
// Must export the SAME interface as the real Cart from checkout-mfe

interface CartProps {
  userId: string
  onCheckout: () => void
}

export default function Cart({ userId, onCheckout }: CartProps) {
  return (
    <div style={{
      padding: 16,
      border: '2px dashed #ccc',
      borderRadius: 8,
      background: '#fafafa',
    }}>
      <p style={{ color: '#999', fontSize: 12, marginBottom: 8 }}>
        [Mock] checkout/Cart
      </p>
      <p>User ID: {userId}</p>
      <button onClick={onCheckout}>Mock Checkout</button>
    </div>
  )
}
```

### Step 3 — Shell component stays identical

```tsx
// shell/src/pages/ShopPage.tsx
// This file NEVER changes — it doesn't know about mocks vs real remotes

const Cart = React.lazy(() => import('checkout/Cart'))

export default function ShopPage() {
  return (
    <React.Suspense fallback={<div>Loading...</div>}>
      <Cart userId="u123" onCheckout={() => alert('checkout!')} />
    </React.Suspense>
  )
}
```

### Running with and without mocks

```bash
# With mocks — ONE terminal, full HMR everywhere (~200MB RAM)
USE_MOCKS=true vite dev

# Without mocks — all remotes must be running
cd checkout-mfe && vite build --watch &
vite preview --port 5001 &
cd shell && vite dev
```

### What mocks give you vs what they can't

| With mocks | Without mocks (real remotes) |
|---|---|
| Full HMR everywhere | HMR only within each app |
| One terminal | 2 processes per remote |
| Fast startup | Slow startup |
| Test shell layout & routing | Test real remote business logic |
| Test prop contracts | Test real API calls |
| Test Suspense/loading states | Test real CSS from remote |
| Low RAM | High RAM |

> **Mental model:** Mocks = unit testing the shell in isolation. Real remotes = integration testing the whole system. Use mocks for daily feature work, spin up real remotes only for pre-release integration testing.

---

## When NOT to Use Microfrontends

Microfrontends add real, permanent complexity to your project. They are the right choice only when the problem they solve is bigger than the cost they introduce.

**Good reasons to use MFEs:**
- You have 3+ independent teams working on the same frontend
- Build times are over 10 minutes and slowing everyone down
- Different parts of the app genuinely need different deploy cadences
- One team wants Vue, another wants React, and that is a firm requirement

**Bad reasons to use MFEs:**
- You want to sound architectural
- You have one team of 5 engineers
- Your app is straightforward with no performance bottlenecks
- You haven't tried modular monolith patterns first

For a single team, a well-structured monolith with clear module boundaries, lazy loading, and good code-splitting will deliver 90% of the benefit with 10% of the complexity.

---

## Quick Reference

```bash
# Start all remotes for local integration testing
cd nav-mfe      && vite build --watch & vite preview --port 5001 &
cd checkout-mfe && vite build --watch & vite preview --port 5002 &
cd shell        && vite dev --port 3000

# Start with mocks (recommended daily workflow)
cd shell && USE_MOCKS=true vite dev --port 3000

# Production build
cd checkout-mfe && vite build && aws s3 sync dist/ s3://cdn/checkout/
cd products-mfe && vite build && aws s3 sync dist/ s3://cdn/products/
cd shell        && vite build && aws s3 sync dist/ s3://cdn/shell/
```

```ts
// Minimal correct shared config — copy this to every MFE and the shell
shared: {
  react:              { singleton: true, requiredVersion: '^18.2.0', eager: true },
  'react-dom':        { singleton: true, requiredVersion: '^18.2.0', eager: true },
  'react-router-dom': { singleton: true, requiredVersion: '^6.0.0' },
}
```

---

*Generated with reference to React source code, Vite documentation, and @originjs/vite-plugin-federation.*
