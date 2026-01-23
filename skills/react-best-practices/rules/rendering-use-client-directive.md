---
title: Understanding 'use client' Directive
impact: HIGH
impactDescription: prevents architectural mistakes in React Server Components
tags: rendering, rsc, server-components, client-components, architecture
---

## Understanding 'use client' Directive

**Impact: HIGH (prevents architectural mistakes in React Server Components)**

**Common misconception**: `'use client'` means the code runs only in the browser.

**Reality**: Client Components still render on the server (SSR), then hydrate on the client. The directive marks the boundary between Server Components and Client Components, not between server and browser execution.

**Incorrect (adding 'use client' to entire page):**

```tsx
'use client'

export default function Page() {
  const [state, setState] = useState()
  // Everything here ships to client, even static content
  return (
    <div>
      <h1>Welcome</h1>
      <p>Static content that could stay on server...</p>
      <InteractiveWidget state={state} />
    </div>
  )
}
```

Making the entire page a Client Component ships unnecessary JavaScript and prevents server-side data fetching.

**Correct (push 'use client' down the tree):**

```tsx
// Page stays as Server Component
export default async function Page() {
  const data = await fetchData() // Can fetch data directly
  return (
    <div>
      <h1>Welcome</h1>
      <p>Static content stays on server</p>
      <InteractiveWidget initialData={data} /> {/* Only this needs 'use client' */}
    </div>
  )
}

// Separate file: InteractiveWidget.tsx
'use client'

export function InteractiveWidget({ initialData }) {
  const [state, setState] = useState(initialData)
  return <button onClick={() => setState(...)}>Click</button>
}
```

### When to Use 'use client'

Add `'use client'` when you need:
- React hooks (`useState`, `useEffect`, etc.)
- Event handlers (`onClick`, `onChange`, etc.)
- Browser APIs (after hydration)
- Class components with lifecycle methods

### When NOT to Use 'use client'

Keep components as Server Components when:
- Only displaying data (no interactivity)
- Fetching and passing data to children
- Using server-only features (database, file system)

### Common Mistake: Thinking 'use client' Skips SSR

**Incorrect (assuming no server render):**

```tsx
'use client'

function Timer() {
  // WRONG assumption: "This won't run on server"
  // REALITY: This WILL run on server during SSR
  const now = new Date()
  return <div>{now.toLocaleString()}</div>
}
```

This causes hydration mismatch because the date differs between server and client.

**Correct (handle client-only values properly):**

```tsx
'use client'

function Timer() {
  const [now, setNow] = useState<Date>()

  useEffect(() => {
    setNow(new Date())
  }, [])

  return <div>{now?.toLocaleString() ?? 'Loading...'}</div>
}
```

### Understanding 'use server'

`'use server'` marks functions as **Server Actions** - functions that execute on the server but can be called from the client:

```tsx
'use server'

export async function submitForm(formData: FormData) {
  // This runs on the server, even when called from a client component
  await db.forms.create({ data: Object.fromEntries(formData) })
}
```

Reference: [React Server Components](https://react.dev/reference/rsc/server-components)
