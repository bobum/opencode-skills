---
name: react-best-practices
description: React performance optimization guidelines from Vercel Engineering. Use when writing, reviewing, or refactoring React components to ensure optimal performance patterns.
license: MIT
metadata:
  author: vercel
  version: "1.0.0"
  source: https://github.com/vercel-labs/agent-skills
  upstream_commit: e23951b8cad2f4b1e7e176c5731127c1263fe86f
  copied_date: "2026-01-31"
  modified: Next.js-specific rules removed (for Vite/React projects)
---

<!--
  UPSTREAM SOURCE: https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices
  Note: Next.js-specific rules have been removed. This version is for Vite/React projects.
-->

# React Best Practices

Performance optimization guide for React applications, adapted from Vercel Engineering. 47 rules across 7 categories, prioritized by impact.

## When to Apply

Reference these guidelines when:
- Writing new React components
- Implementing client-side data fetching
- Reviewing code for performance issues
- Refactoring existing React code
- Optimizing bundle size or load times

---

## 1. Eliminating Waterfalls (CRITICAL)

### Defer Await Until Needed

Move `await` operations into the branches where they're actually used to avoid blocking code paths that don't need them.

**Incorrect (blocks both branches):**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  const userData = await fetchUserData(userId)

  if (skipProcessing) {
    return { skipped: true }
  }

  return processUserData(userData)
}
```

**Correct (only blocks when needed):**

```typescript
async function handleRequest(userId: string, skipProcessing: boolean) {
  if (skipProcessing) {
    return { skipped: true }
  }

  const userData = await fetchUserData(userId)
  return processUserData(userData)
}
```

### Promise.all() for Independent Operations

When async operations have no interdependencies, execute them concurrently.

**Incorrect (sequential, 3 round trips):**

```typescript
const user = await fetchUser()
const posts = await fetchPosts()
const comments = await fetchComments()
```

**Correct (parallel, 1 round trip):**

```typescript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
])
```

### Dependency-Based Parallelization

For operations with partial dependencies, use `better-all` or create promises upfront.

**Correct (config and profile run in parallel):**

```typescript
const userPromise = fetchUser()
const profilePromise = userPromise.then(user => fetchProfile(user.id))

const [user, config, profile] = await Promise.all([
  userPromise,
  fetchConfig(),
  profilePromise
])
```

---

## 2. Bundle Size Optimization (CRITICAL)

### Avoid Barrel File Imports

Import directly from source files instead of barrel files to avoid loading thousands of unused modules.

**Incorrect (imports entire library):**

```tsx
import { Check, X, Menu } from 'lucide-react'
// Loads 1,583 modules, takes ~2.8s extra in dev
```

**Correct (imports only what you need):**

```tsx
import Check from 'lucide-react/dist/esm/icons/check'
import X from 'lucide-react/dist/esm/icons/x'
import Menu from 'lucide-react/dist/esm/icons/menu'
// Loads only 3 modules (~2KB vs ~1MB)
```

Libraries commonly affected: `lucide-react`, `@mui/material`, `@mui/icons-material`, `lodash`, `date-fns`.

### Conditional Module Loading

Load large data or modules only when a feature is activated.

```tsx
function AnimationPlayer({ enabled }: { enabled: boolean }) {
  const [frames, setFrames] = useState<Frame[] | null>(null)

  useEffect(() => {
    if (enabled && !frames) {
      import('./animation-frames.js')
        .then(mod => setFrames(mod.frames))
    }
  }, [enabled, frames])

  if (!frames) return <Skeleton />
  return <Canvas frames={frames} />
}
```

### Defer Non-Critical Third-Party Libraries

Analytics, logging, and error tracking don't block user interaction. Load them after hydration using lazy loading.

### Preload Based on User Intent

Preload heavy bundles before they're needed to reduce perceived latency.

```tsx
function EditorButton({ onClick }: { onClick: () => void }) {
  const preload = () => { void import('./monaco-editor') }

  return (
    <button onMouseEnter={preload} onFocus={preload} onClick={onClick}>
      Open Editor
    </button>
  )
}
```

---

## 3. Client-Side Data Fetching (MEDIUM-HIGH)

### Use SWR for Automatic Deduplication

SWR enables request deduplication, caching, and revalidation across component instances.

```tsx
import useSWR from 'swr'

function UserList() {
  const { data: users } = useSWR('/api/users', fetcher)
}
```

### Deduplicate Global Event Listeners

Use `useSWRSubscription()` to share global event listeners across component instances.

### Use Passive Event Listeners for Scrolling

Add `{ passive: true }` to touch and wheel event listeners to enable immediate scrolling.

```typescript
document.addEventListener('touchstart', handleTouch, { passive: true })
document.addEventListener('wheel', handleWheel, { passive: true })
```

### Version and Minimize localStorage Data

Add version prefix to keys and store only needed fields.

```typescript
const VERSION = 'v2'

function saveConfig(config: { theme: string; language: string }) {
  try {
    localStorage.setItem(`userConfig:${VERSION}`, JSON.stringify(config))
  } catch {}
}
```

---

## 4. Re-render Optimization (MEDIUM)

### Defer State Reads to Usage Point

Don't subscribe to dynamic state (searchParams, localStorage) if you only read it inside callbacks.

**Incorrect (subscribes to all searchParams changes):**

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const searchParams = useSearchParams()
  const handleShare = () => {
    const ref = searchParams.get('ref')
    shareChat(chatId, { ref })
  }
  return <button onClick={handleShare}>Share</button>
}
```

**Correct (reads on demand):**

```tsx
function ShareButton({ chatId }: { chatId: string }) {
  const handleShare = () => {
    const params = new URLSearchParams(window.location.search)
    const ref = params.get('ref')
    shareChat(chatId, { ref })
  }
  return <button onClick={handleShare}>Share</button>
}
```

### Extract to Memoized Components

Extract expensive work into memoized components to enable early returns.

```tsx
const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  const id = useMemo(() => computeAvatarId(user), [user])
  return <Avatar id={id} />
})

function Profile({ user, loading }: Props) {
  if (loading) return <Skeleton />
  return <div><UserAvatar user={user} /></div>
}
```

### Extract Default Non-primitive Props to Constants

```tsx
const NOOP = () => {};

const UserAvatar = memo(function UserAvatar({ onClick = NOOP }: { onClick?: () => void }) {
  // ...
})
```

### Narrow Effect Dependencies

Specify primitive dependencies instead of objects to minimize effect re-runs.

```tsx
// Incorrect
useEffect(() => { console.log(user.id) }, [user])

// Correct
useEffect(() => { console.log(user.id) }, [user.id])
```

### Calculate Derived State During Rendering

If a value can be computed from current props/state, derive it during render.

```tsx
// Incorrect
const [fullName, setFullName] = useState('')
useEffect(() => {
  setFullName(firstName + ' ' + lastName)
}, [firstName, lastName])

// Correct
const fullName = firstName + ' ' + lastName
```

### Use Functional setState Updates

Prevents stale closures and creates stable callback references.

```tsx
const addItems = useCallback((newItems: Item[]) => {
  setItems(curr => [...curr, ...newItems])
}, [])  // No dependencies needed
```

### Use Lazy State Initialization

Pass a function to `useState` for expensive initial values.

```tsx
// Incorrect (runs on every render)
const [searchIndex] = useState(buildSearchIndex(items))

// Correct (runs only once)
const [searchIndex] = useState(() => buildSearchIndex(items))
```

### Use Transitions for Non-Urgent Updates

```tsx
import { startTransition } from 'react'

const handler = () => {
  startTransition(() => setScrollY(window.scrollY))
}
```

### Use useRef for Transient Values

When a value changes frequently and you don't want a re-render on every update.

```tsx
function Tracker() {
  const lastXRef = useRef(0)
  const dotRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const onMove = (e: MouseEvent) => {
      lastXRef.current = e.clientX
      if (dotRef.current) {
        dotRef.current.style.transform = `translateX(${e.clientX}px)`
      }
    }
    window.addEventListener('mousemove', onMove)
    return () => window.removeEventListener('mousemove', onMove)
  }, [])
}
```

### Put Interaction Logic in Event Handlers

If a side effect is triggered by a specific user action, run it in that event handler.

```tsx
// Incorrect
useEffect(() => {
  if (submitted) { post('/api/register') }
}, [submitted])

// Correct
function handleSubmit() {
  post('/api/register')
}
```

### Don't Wrap Simple Expressions in useMemo

When an expression is simple and has a primitive result type, skip useMemo.

```tsx
// Incorrect
const isLoading = useMemo(() => {
  return user.isLoading || notifications.isLoading
}, [user.isLoading, notifications.isLoading])

// Correct
const isLoading = user.isLoading || notifications.isLoading
```

---

## 5. Rendering Performance (MEDIUM)

### CSS content-visibility for Long Lists

Apply `content-visibility: auto` to defer off-screen rendering.

```css
.message-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
```

### Hoist Static JSX Elements

Extract static JSX outside components to avoid re-creation.

```tsx
const loadingSkeleton = (
  <div className="animate-pulse h-20 bg-gray-200" />
)

function Container() {
  return <div>{loading && loadingSkeleton}</div>
}
```

### Animate SVG Wrapper Instead of SVG Element

Wrap SVG in a `<div>` and animate the wrapper for hardware acceleration.

```tsx
// Correct
<div className="animate-spin">
  <svg>...</svg>
</div>
```

### Use Explicit Conditional Rendering

Use ternary operators instead of `&&` when the condition can be `0` or `NaN`.

```tsx
// Incorrect (renders "0" when count is 0)
{count && <Badge>{count}</Badge>}

// Correct
{count > 0 ? <Badge>{count}</Badge> : null}
```

### Use Activity Component for Show/Hide

Use React's `<Activity>` to preserve state/DOM for components that toggle visibility.

```tsx
import { Activity } from 'react'

function Dropdown({ isOpen }: Props) {
  return (
    <Activity mode={isOpen ? 'visible' : 'hidden'}>
      <ExpensiveMenu />
    </Activity>
  )
}
```

### Prevent Hydration Mismatch Without Flickering

For client-only data (localStorage, cookies), use inline scripts to set values before React hydrates.

### Use useTransition Over Manual Loading States

```tsx
const [isPending, startTransition] = useTransition()

const handleSearch = (value: string) => {
  setQuery(value)
  startTransition(async () => {
    const data = await fetchResults(value)
    setResults(data)
  })
}
```

---

## 6. JavaScript Performance (LOW-MEDIUM)

### Avoid Layout Thrashing

Batch style writes, then read. Don't interleave reads and writes.

```typescript
// Correct: batch writes, then read
element.style.width = '100px'
element.style.height = '200px'
const { width, height } = element.getBoundingClientRect()
```

### Cache Repeated Function Calls

Use a module-level Map to cache function results.

```typescript
const slugifyCache = new Map<string, string>()

function cachedSlugify(text: string): string {
  if (slugifyCache.has(text)) return slugifyCache.get(text)!
  const result = slugify(text)
  slugifyCache.set(text, result)
  return result
}
```

### Cache Property Access in Loops

```typescript
const value = obj.config.settings.value
const len = arr.length
for (let i = 0; i < len; i++) {
  process(value)
}
```

### Cache Storage API Calls

localStorage, sessionStorage, and document.cookie are synchronous and expensive. Cache reads in memory.

### Combine Multiple Array Iterations

```typescript
// Incorrect (3 iterations)
const admins = users.filter(u => u.isAdmin)
const testers = users.filter(u => u.isTester)

// Correct (1 iteration)
const admins: User[] = []
const testers: User[] = []
for (const user of users) {
  if (user.isAdmin) admins.push(user)
  if (user.isTester) testers.push(user)
}
```

### Early Return from Functions

Return early when result is determined to skip unnecessary processing.

### Hoist RegExp Creation

Don't create RegExp inside render. Hoist to module scope or memoize.

```tsx
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
```

### Build Index Maps for Repeated Lookups

```typescript
const userById = new Map(users.map(u => [u.id, u]))
return orders.map(order => ({
  ...order,
  user: userById.get(order.userId)
}))
```

### Early Length Check for Array Comparisons

```typescript
function hasChanges(current: string[], original: string[]) {
  if (current.length !== original.length) return true
  // ... then do expensive comparison
}
```

### Use Loop for Min/Max Instead of Sort

```typescript
function getLatestProject(projects: Project[]) {
  let latest = projects[0]
  for (let i = 1; i < projects.length; i++) {
    if (projects[i].updatedAt > latest.updatedAt) {
      latest = projects[i]
    }
  }
  return latest
}
```

### Use Set/Map for O(1) Lookups

```typescript
const allowedIds = new Set(['a', 'b', 'c'])
items.filter(item => allowedIds.has(item.id))
```

### Use toSorted() Instead of sort() for Immutability

`.sort()` mutates the array. Use `.toSorted()` to create a new sorted array.

```typescript
const sorted = users.toSorted((a, b) => a.name.localeCompare(b.name))
```

---

## 7. Advanced Patterns (LOW)

### Store Event Handlers in Refs

Store callbacks in refs when used in effects that shouldn't re-subscribe on callback changes.

```tsx
function useWindowEvent(event: string, handler: (e) => void) {
  const handlerRef = useRef(handler)
  useEffect(() => { handlerRef.current = handler }, [handler])

  useEffect(() => {
    const listener = (e) => handlerRef.current(e)
    window.addEventListener(event, listener)
    return () => window.removeEventListener(event, listener)
  }, [event])
}
```

### Initialize App Once, Not Per Mount

Do not put app-wide initialization inside `useEffect([])`. Use a module-level guard.

```tsx
let didInit = false

function Comp() {
  useEffect(() => {
    if (didInit) return
    didInit = true
    loadFromStorage()
    checkAuthToken()
  }, [])
}
```

### useEffectEvent for Stable Callback Refs

Access latest values in callbacks without adding them to dependency arrays.

```tsx
import { useEffectEvent } from 'react';

function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState('')
  const onSearchEvent = useEffectEvent(onSearch)

  useEffect(() => {
    const timeout = setTimeout(() => onSearchEvent(query), 300)
    return () => clearTimeout(timeout)
  }, [query])
}
```
