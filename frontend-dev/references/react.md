# React Reference

Covers React 19.2, Next.js 16, React Router v7, and the modern React ecosystem. No class components.

## Project Setup

Prefer **Next.js** for new apps (SSR, RSC, file-based routing, image optimization, Turbopack default).
Use **React Router v7** (framework mode) for full-stack apps where you want web-standard loaders/actions without Vercel-centric patterns — this is the direct successor to Remix, which merged into React Router v7 in late 2024.
Use **Vite + React Router v7** (library mode) for pure SPAs that don't need SSR.
**TanStack Start** is a credible alternative for TypeScript-first apps that want TanStack Router's type safety with a full-stack story.

Note on Remix: as a standalone React framework, Remix no longer exists — React Router v7 framework mode is what it became. A separate project called "Remix 3" exists but is a non-React, imperative framework built on a Preact fork; treat it as an unrelated project unless you're specifically evaluating it.

```bash
# Next.js 16 (Turbopack, App Router, TS, Tailwind by default)
pnpm create next-app@latest my-app --yes

# React Router v7 framework mode
pnpm create react-router@latest my-app

# Vite SPA
pnpm create vite@latest my-app -- --template react-ts
```

---

## Component Patterns

### Function components only

```typescript
// Standard component
interface ButtonProps {
  label: string
  variant?: 'primary' | 'secondary' | 'ghost'
  onClick: () => void
  disabled?: boolean
}

export function Button({ label, variant = 'primary', onClick, disabled = false }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant }))}
      onClick={onClick}
      disabled={disabled}
    >
      {label}
    </button>
  )
}
```

### Composition with children

Prefer `children` and slot patterns over heavily prop-drilled components:

```typescript
interface CardProps {
  children: React.ReactNode
  className?: string
}

export function Card({ children, className }: CardProps) {
  return <div className={cn('rounded-lg border p-4', className)}>{children}</div>
}

// Usage — composable, no prop explosion
<Card>
  <Card.Header>Title</Card.Header>
  <Card.Body>Content</Card.Body>
</Card>
```

### Custom hooks for logic extraction

Extract anything non-trivial from components into hooks. Hooks are testable in isolation.

```typescript
// hooks/useDebounce.ts
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])
  return debouncedValue
}
```

---

## State Management

Choose based on complexity:

| Scenario | Solution |
|---|---|
| Local UI state | `useState`, `useReducer` |
| Shared UI state (theme, modals) | Context + `useReducer` |
| Server state (fetching, caching) | TanStack Query (React Query) |
| Complex global client state | Zustand |
| Large-scale with devtools / time-travel | Redux Toolkit |

**Default for most apps**: `useState` + TanStack Query. Don't add Zustand or Redux until you have a clear need.

Context API is for low-update-frequency data (auth state, theme, locale). Not for frequently updating state — every consumer re-renders on context change.

### TanStack Query pattern

```typescript
// hooks/useUser.ts
export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => userService.getById(id),
    staleTime: 5 * 60 * 1000, // 5 minutes
  })
}

// Mutation with optimistic update
export function useUpdateUser() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: userService.update,
    onMutate: async (updatedUser) => {
      await queryClient.cancelQueries({ queryKey: ['user', updatedUser.id] })
      const previous = queryClient.getQueryData(['user', updatedUser.id])
      queryClient.setQueryData(['user', updatedUser.id], updatedUser)
      return { previous }
    },
    onError: (_, updatedUser, context) => {
      queryClient.setQueryData(['user', updatedUser.id], context?.previous)
    },
    onSettled: (_, __, updatedUser) => {
      queryClient.invalidateQueries({ queryKey: ['user', updatedUser.id] })
    },
  })
}
```

---

## React 19.2 Features

React 19.2 shipped October 2025. Key additions over React 19.0/19.1:

### React Compiler (stable, 1.0 released)

The React Compiler automatically memoizes components. It reached 1.0 in late 2025 and is stable in Next.js 16 (opt-in via `reactCompiler: true` in `next.config.ts`). In projects where the compiler is enabled, **do not manually add `useMemo`, `useCallback`, or `React.memo`** unless you have a profiler-measured reason — the compiler handles it and manual memoization can interfere. Without the compiler, the old manual rules still apply.

### `<Activity>` — hide UI while preserving state

Replaces the `{isVisible && <Component />}` pattern when you want to hide a subtree without losing its state:

```typescript
import { Activity } from 'react'

function TabPanel({ activeTab }: { activeTab: string }) {
  return (
    <>
      <Activity mode={activeTab === 'home' ? 'visible' : 'hidden'}>
        <Home />
      </Activity>
      <Activity mode={activeTab === 'contact' ? 'visible' : 'hidden'}>
        <Contact />
      </Activity>
    </>
  )
}
```

When hidden, effects unmount but DOM state (form inputs, scroll position) is preserved. Use it for tabs, modals, and pre-rendering content the user is likely to navigate to next.

### `useEffectEvent` — stable callbacks inside effects

Fixes the stale-closure problem without re-running the effect:

```typescript
function ChatRoom({ roomId, theme }: Props) {
  const onConnect = useEffectEvent(() => {
    showNotification('Connected!', theme) // Always sees latest theme
  })

  useEffect(() => {
    const connection = createConnection(roomId)
    connection.on('connected', onConnect)
    return () => connection.disconnect()
  }, [roomId]) // theme is NOT in deps — onConnect handles that
}
```

Only call `useEffectEvent` callbacks from inside effects, not from render or event handlers.

### `<ViewTransition>` — animated UI transitions

Wrap elements that should animate across state changes. Only activates inside `startTransition`, `<Suspense>`, or `useDeferredValue` updates:

```typescript
import { ViewTransition, startTransition } from 'react'

function Gallery() {
  const [selected, setSelected] = useState<string | null>(null)
  return (
    <>
      {items.map(item => (
        <ViewTransition key={item.id}>
          <img src={item.src} onClick={() => startTransition(() => setSelected(item.id))} />
        </ViewTransition>
      ))}
    </>
  )
}
```

Backed by the browser's `startViewTransition` API. Next.js uses `<Activity>` internally for client-side navigation when Cache Components is enabled.

### Server Components (Next.js App Router)

Default components in the App Router are **React Server Components (RSC)**. They run on the server, have no client JS bundle cost, and can directly access databases/services.

```typescript
// app/users/page.tsx — Server Component (default)
export default async function UsersPage() {
  const users = await db.user.findMany() // Direct DB access — no API call needed
  return <UserList users={users} />
}
```

Add `'use client'` only when you need interactivity (event handlers, hooks, browser APIs):

```typescript
'use client'
export function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [value, setValue] = useState('')
  return <input value={value} onChange={e => { setValue(e.target.value); onSearch(e.target.value) }} />
}
```

**Rule**: Push `'use client'` as far down the component tree as possible. Keep data fetching in Server Components.

**Security note**: Three critical CVEs were disclosed against the RSC protocol in late 2025 (CVE-2025-66478 CVSS 10.0, CVE-2025-55184, CVE-2025-55183). Keep Next.js patched to 16.x and audit dependencies if the project targets self-hosted RSC.

### Server Actions

```typescript
// actions/user.ts
'use server'
export async function updateUser(id: string, data: Partial<User>) {
  const validated = UserUpdateSchema.parse(data)
  return db.user.update({ where: { id }, data: validated })
}

// Component usage
<form action={updateUser.bind(null, userId)}>
  <input name="name" />
  <button type="submit">Save</button>
</form>
```

### Next.js 16 specifics

- **Turbopack is stable and default** for both `next dev` and `next build`. No `--turbopack` flag needed. Opt out with `--webpack` only if you have an unmigrated webpack config.
- **Async `params`/`searchParams` fully enforced**: `params` is a `Promise`, must be `await`ed (or read with `use()` in client components). The sync compatibility shim from Next.js 15 is gone.
- **Cache Components** (formerly PPR): the new programming model for mixing static shells with streaming dynamic fragments. Controlled with `"use cache"` directive.
- **`create-next-app` ships `AGENTS.md` / `CLAUDE.md`** by default to give coding agents up-to-date Next.js context.
- **Node 20.9+** is the minimum.

---

## Rendering Strategy (Next.js)

| Strategy | When to use |
|---|---|
| SSR (dynamic) | User-specific, real-time, auth-gated pages |
| SSG (static) | Marketing pages, docs, content that rarely changes |
| ISR | Content that changes periodically (e.g. hourly) |
| Cache Components | Pages that mix a static shell with streaming dynamic fragments — the Next.js 16 successor to Partial Prerendering, controlled via `"use cache"` |
| CSR | Dashboards, internal tools where SEO doesn't matter |

Prefer SSR, SSG, or Cache Components by default. Only go CSR when there's a specific reason.

---

## Performance

### Code splitting

```typescript
// Lazy load heavy components
const HeavyChart = lazy(() => import('./HeavyChart'))

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <HeavyChart />
    </Suspense>
  )
}
```

### Image optimization (Next.js)

```typescript
import Image from 'next/image'

// Always use next/image — handles WebP conversion, lazy loading, CLS prevention
<Image
  src="/hero.jpg"
  width={1200}
  height={600}
  alt="Hero image"
  priority // Add for LCP images — disables lazy loading
/>
```

### Avoid unnecessary re-renders

With the React Compiler active, focus on:
- Keeping state as local as possible
- Avoiding object/array literals as default prop values (new reference each render)
- Not creating functions inline when passing to non-compiled third-party components

---

## File Structure (feature-based)

```
src/
  app/                    # Next.js App Router pages
    (auth)/
      login/page.tsx
    dashboard/
      page.tsx
      layout.tsx
  features/
    users/
      components/
        UserCard.tsx
        UserCard.test.tsx
      hooks/
        useUser.ts
        useUser.test.ts
      services/
        userService.ts     # API calls, data transformation
      types.ts             # Feature-specific types
  components/
    ui/                    # Shared primitives (Button, Input, Modal)
    layout/                # Header, Sidebar, Footer
  lib/
    api.ts                 # Axios/fetch instance with auth interceptors
    queryClient.ts         # TanStack Query configuration
  hooks/                   # Shared hooks (useDebounce, useLocalStorage)
  types/                   # Global shared types
```

---

## Testing (React)

```typescript
// UserCard.test.tsx
import { render, screen, userEvent } from '@testing-library/react'

describe('UserCard', () => {
  it('displays user name and email', () => {
    render(<UserCard user={{ id: '1', name: 'Alice', email: 'alice@example.com' }} />)
    expect(screen.getByText('Alice')).toBeInTheDocument()
    expect(screen.getByText('alice@example.com')).toBeInTheDocument()
  })

  it('calls onEdit when edit button is clicked', async () => {
    const onEdit = vi.fn()
    render(<UserCard user={mockUser} onEdit={onEdit} />)
    await userEvent.click(screen.getByRole('button', { name: /edit/i }))
    expect(onEdit).toHaveBeenCalledWith(mockUser.id)
  })
})
```

Use `@testing-library/react`. Test what users see and do, not internal state or implementation details.

---

## Key Libraries (2026 defaults)

| Concern | Library |
|---|---|
| Server state | TanStack Query v5 |
| Client state | Zustand |
| Forms | React Hook Form + Zod |
| Styling | Tailwind CSS v4 |
| UI primitives | shadcn/ui (Radix-based) |
| Animation | Motion (formerly Framer Motion) |
| Tables | TanStack Table |
| Routing (SPA) | React Router v7 or TanStack Router |
| Date | date-fns or Temporal (where available) |
| Icons | Lucide React |
| Testing | Vitest + Testing Library + Playwright |

Notes: TanStack Query v6 exists but is currently Svelte-only — React stays on v5. TanStack Router is the stronger pick when compile-time route type safety matters; React Router v7 wins on ecosystem size and SSR maturity.
