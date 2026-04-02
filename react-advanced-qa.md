# React Advanced Interview Prep (Architect Edition)

This guide focuses on modern React patterns (React 18/19 era), production-grade tradeoffs, and implementation depth expected in senior interviews.

## 1) Q: How does concurrent rendering improve UX, and when would you use useTransition?

**Answer:**
Concurrent rendering lets React prioritize urgent updates (typing, clicks) over non-urgent work (large list filtering, route updates). `useTransition` marks updates as low priority so UI stays responsive.

~~~tsx
import { useMemo, useState, useTransition } from 'react';

type Item = { id: number; name: string };

export function SearchList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [filter, setFilter] = useState('');

  function onChange(value: string) {
    setQuery(value); // urgent: keep input responsive
    startTransition(() => {
      setFilter(value); // non-urgent: can be interrupted
    });
  }

  const filtered = useMemo(() => {
    const q = filter.toLowerCase();
    return items.filter((x) => x.name.toLowerCase().includes(q));
  }, [items, filter]);

  return (
    <div>
      <input value={query} onChange={(e) => onChange(e.target.value)} />
      {isPending && <p>Updating results...</p>}
      <ul>{filtered.map((x) => <li key={x.id}>{x.name}</li>)}</ul>
    </div>
  );
}
~~~

**Key implementation point:** Separate urgent and non-urgent state. Don’t wrap every update in transitions.

---

## 2) Q: Difference between useTransition and useDeferredValue?

**Answer:**
- `useTransition`: defer a state update you trigger.
- `useDeferredValue`: defer consumption of a value you already have.

~~~tsx
import { useDeferredValue, useMemo, useState } from 'react';

export function DeferredSearch({ data }: { data: string[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // lagging copy for expensive work

  const result = useMemo(() => {
    const q = deferredQuery.toLowerCase();
    return data.filter((x) => x.toLowerCase().includes(q));
  }, [data, deferredQuery]);

  const isStale = query !== deferredQuery;

  return (
    <section>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isStale && <small>Refreshing...</small>}
      <pre>{JSON.stringify(result.slice(0, 10), null, 2)}</pre>
    </section>
  );
}
~~~

**Key implementation point:** Great when parent owns `query`, and child should process it lazily.

---

## 3) Q: How do you structure Suspense for data-fetching boundaries?

**Answer:**
Use smaller Suspense boundaries near slow components instead of one global boundary. Pair with Error Boundaries to isolate failures.

~~~tsx
import { Suspense } from 'react';

function UserPanel() {
  // Imagine this component reads async data via framework/cache.
  return <div>User panel loaded</div>;
}

function ActivityFeed() {
  return <div>Activity feed loaded</div>;
}

export function Dashboard() {
  return (
    <div>
      <Suspense fallback={<p>Loading user...</p>}>
        <UserPanel />
      </Suspense>

      <Suspense fallback={<p>Loading activity...</p>}>
        <ActivityFeed />
      </Suspense>
    </div>
  );
}
~~~

**Key implementation point:** Boundary granularity controls perceived performance.

---

## 4) Q: Explain the use hook and where it fits.

**Answer:**
`use` consumes a Promise (or context-like resource in supported environments) during render and integrates with Suspense. It reduces effect-based fetch boilerplate in compatible stacks.

~~~tsx
import { Suspense, use } from 'react';

async function fetchProfile(id: string) {
  const res = await fetch(`/api/profile/${id}`);
  if (!res.ok) throw new Error('Failed profile fetch');
  return res.json() as Promise<{ name: string; role: string }>;
}

function ProfileDetails({ profilePromise }: { profilePromise: Promise<{ name: string; role: string }> }) {
  const profile = use(profilePromise); // suspends until promise resolves
  return <h2>{profile.name} - {profile.role}</h2>;
}

export function Profile({ id }: { id: string }) {
  const profilePromise = fetchProfile(id);

  return (
    <Suspense fallback={<p>Loading profile...</p>}>
      <ProfileDetails profilePromise={profilePromise} />
    </Suspense>
  );
}
~~~

**Key implementation point:** Avoid creating new Promises each render unless your framework cache/dedupe layer handles it.

---

## 5) Q: How do optimistic UI flows work with useOptimistic?

**Answer:**
`useOptimistic` lets you apply temporary client-side state immediately, then reconcile with server response.

~~~tsx
import { useOptimistic, useState } from 'react';

type Message = { id: string; text: string; pending?: boolean };

async function postMessage(text: string): Promise<Message> {
  const res = await fetch('/api/messages', {
    method: 'POST',
    body: JSON.stringify({ text }),
    headers: { 'Content-Type': 'application/json' },
  });
  if (!res.ok) throw new Error('Failed to send');
  return res.json();
}

export function ChatBox({ initial }: { initial: Message[] }) {
  const [messages, setMessages] = useState(initial);
  const [optimistic, addOptimistic] = useOptimistic(
    messages,
    (state, text: string) => [
      ...state,
      { id: `tmp-${Date.now()}`, text, pending: true }, // local optimistic item
    ]
  );

  async function send(text: string) {
    addOptimistic(text);
    try {
      const saved = await postMessage(text);
      setMessages((curr) => [...curr, saved]);
    } catch {
      // In real apps, surface error toast and refresh list.
    }
  }

  return (
    <div>
      <button onClick={() => send('Hello')}>Send</button>
      <ul>
        {optimistic.map((m) => (
          <li key={m.id}>{m.text} {m.pending ? '(sending...)' : ''}</li>
        ))}
      </ul>
    </div>
  );
}
~~~

**Key implementation point:** Keep reconciliation strategy explicit (replace temp IDs, refetch, or append confirmed data).

---

## 6) Q: How do useActionState and form actions simplify async form handling?

**Answer:**
They centralize pending/error/result state around a form action and reduce manual state wiring.

~~~tsx
import { useActionState } from 'react';

type FormState = { ok: boolean; message: string };

async function subscribeAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const email = String(formData.get('email') || '');
  if (!email.includes('@')) return { ok: false, message: 'Invalid email' };

  // Simulate API
  await new Promise((r) => setTimeout(r, 500));
  return { ok: true, message: `Subscribed: ${email}` };
}

export function NewsletterForm() {
  const [state, formAction, isPending] = useActionState(subscribeAction, {
    ok: false,
    message: '',
  });

  return (
    <form action={formAction}>
      <input name="email" placeholder="you@company.com" />
      <button disabled={isPending}>{isPending ? 'Submitting...' : 'Subscribe'}</button>
      <p role="status">{state.message}</p>
    </form>
  );
}
~~~

**Key implementation point:** Keep action pure and deterministic from `(prevState, formData) => nextState`.

---

## 7) Q: How do you prevent unnecessary re-renders in large trees?

**Answer:**
Use memoization where measurement proves gains: stable props, `React.memo`, `useMemo`, `useCallback`, and state locality.

~~~tsx
import { memo, useCallback, useState } from 'react';

const ExpensiveChild = memo(function ExpensiveChild({ onSelect }: { onSelect: (id: number) => void }) {
  // Heavy render work could live here.
  return <button onClick={() => onSelect(42)}>Select 42</button>;
});

export function Parent() {
  const [count, setCount] = useState(0);

  const onSelect = useCallback((id: number) => {
    console.log('Selected', id);
  }, []); // stable function identity avoids child re-render

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ExpensiveChild onSelect={onSelect} />
    </div>
  );
}
~~~

**Key implementation point:** Prefer architectural fixes (state split, component boundaries) before adding memo everywhere.

---

## 8) Q: What is the right pattern for Context at scale?

**Answer:**
Avoid monolithic context values. Split contexts by update frequency and concern to reduce cascading renders.

~~~tsx
import { createContext, useContext, useMemo, useState } from 'react';

const ThemeContext = createContext<{ theme: 'light' | 'dark'; toggle: () => void } | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const value = useMemo(() => ({
    theme,
    toggle: () => setTheme((t) => (t === 'light' ? 'dark' : 'light')),
  }), [theme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}
~~~

**Key implementation point:** For high-frequency global updates, consider external stores with selectors.

---

## 9) Q: How do you integrate an external store safely with React?

**Answer:**
Use `useSyncExternalStore` for concurrent-safe subscriptions.

~~~tsx
import { useSyncExternalStore } from 'react';

type CounterStore = {
  getSnapshot: () => number;
  subscribe: (cb: () => void) => () => void;
  increment: () => void;
};

const store: CounterStore = (() => {
  let value = 0;
  const listeners = new Set<() => void>();

  return {
    getSnapshot: () => value,
    subscribe: (cb) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
    increment: () => {
      value += 1;
      listeners.forEach((l) => l());
    },
  };
})();

export function CounterFromStore() {
  const count = useSyncExternalStore(store.subscribe, store.getSnapshot);

  return <button onClick={store.increment}>Store Count: {count}</button>;
}
~~~

**Key implementation point:** This avoids tearing issues under concurrent rendering.

---

## 10) Q: How do you design resilient Error Boundaries?

**Answer:**
Use Error Boundaries at route/widget boundaries. Provide fallback with reset affordances.

~~~tsx
import React from 'react';

type State = { hasError: boolean };

export class WidgetErrorBoundary extends React.Component<React.PropsWithChildren, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: unknown) {
    // Send to monitoring service (Sentry, Datadog, etc.)
    console.error('Widget crashed', error);
  }

  render() {
    if (this.state.hasError) {
      return <button onClick={() => this.setState({ hasError: false })}>Retry Widget</button>;
    }
    return this.props.children;
  }
}
~~~

**Key implementation point:** Error Boundaries catch render/lifecycle errors, not async event handler errors.

---

## 11) Q: How do you handle stale closures in hooks?

**Answer:**
If effect logic needs latest values, include dependencies or move mutable values to refs with controlled access.

~~~tsx
import { useEffect, useRef, useState } from 'react';

export function Timer() {
  const [seconds, setSeconds] = useState(0);
  const latest = useRef(seconds);

  useEffect(() => {
    latest.current = seconds; // keep ref synced with latest state
  }, [seconds]);

  useEffect(() => {
    const id = setInterval(() => {
      // Safe read from ref avoids stale closure
      console.log('Latest seconds:', latest.current);
      setSeconds((s) => s + 1);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return <p>{seconds}s</p>;
}
~~~

**Key implementation point:** Prefer dependency-correct effects first; refs are an escape hatch.

---

## 12) Q: What are key patterns for Server Components architecture (framework-based)?

**Answer:**
- Keep heavy data joins and secrets on server components.
- Use client components only for interactivity.
- Pass serializable props across the server-client boundary.

~~~tsx
// Server component (framework-specific): data fetch on server
export default async function ProductPage({ id }: { id: string }) {
  const product = await fetch(`https://api.example.com/products/${id}`, {
    cache: 'no-store',
  }).then((r) => r.json());

  return (
    <main>
      <h1>{product.name}</h1>
      <AddToCartButton productId={product.id} />
    </main>
  );
}

// Client component: interactivity only
'use client';
import { useState } from 'react';

function AddToCartButton({ productId }: { productId: string }) {
  const [busy, setBusy] = useState(false);
  return (
    <button
      disabled={busy}
      onClick={async () => {
        setBusy(true);
        await fetch('/api/cart', { method: 'POST', body: JSON.stringify({ productId }) });
        setBusy(false);
      }}
    >
      {busy ? 'Adding...' : 'Add to cart'}
    </button>
  );
}
~~~

**Key implementation point:** Treat server/client boundary as an architectural contract.

---

## 13) Q: How do you test advanced React behavior, not just static rendering?

**Answer:**
Test user-observable behavior and async transitions with React Testing Library.

~~~tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchList } from './SearchList';

test('keeps input responsive while list updates', async () => {
  const items = Array.from({ length: 5000 }, (_, i) => ({ id: i, name: `item-${i}` }));
  render(<SearchList items={items} />);

  const input = screen.getByRole('textbox');
  await userEvent.type(input, 'item-49');

  // User-facing assertion
  expect((input as HTMLInputElement).value).toBe('item-49');
});
~~~

**Key implementation point:** Assertions should reflect UX guarantees, not internal implementation details.

---

## 14) Q: Explain a production-ready data fetching strategy in React apps.

**Answer:**
Use a layered approach:
- Route/server-level fetch for primary page data.
- Client cache library (TanStack Query/SWR) for interactive refetching and mutation.
- Suspense boundaries for progressive rendering.
- Optimistic updates for mutation-heavy surfaces.

~~~tsx
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';

type Todo = { id: string; title: string };

function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: async () => {
      const res = await fetch('/api/todos');
      return (await res.json()) as Todo[];
    },
  });
}

export function TodoList() {
  const client = useQueryClient();
  const { data = [] } = useTodos();

  const createTodo = useMutation({
    mutationFn: async (title: string) => {
      const res = await fetch('/api/todos', { method: 'POST', body: JSON.stringify({ title }) });
      return (await res.json()) as Todo;
    },
    onMutate: async (title) => {
      await client.cancelQueries({ queryKey: ['todos'] });
      const prev = client.getQueryData<Todo[]>(['todos']) || [];
      client.setQueryData<Todo[]>(['todos'], [...prev, { id: `tmp-${Date.now()}`, title }]);
      return { prev };
    },
    onError: (_err, _title, ctx) => {
      if (ctx?.prev) client.setQueryData(['todos'], ctx.prev);
    },
    onSettled: () => {
      client.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <section>
      <button onClick={() => createTodo.mutate('New task')}>Add</button>
      <ul>{data.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
    </section>
  );
}
~~~

**Key implementation point:** Reliable optimistic flows always include rollback and revalidation.

---

## 15) Q: Architect a React frontend for scale. What principles would you enforce?

**Answer:**
1. Slice by domain features, not by technical layer only.
2. Keep state close to usage; elevate only when needed.
3. Introduce clear boundaries: route, data, presentation, shared UI.
4. Prefer composition over inheritance and giant prop objects.
5. Enforce performance budgets (bundle size, TTI, interaction latency).
6. Codify patterns with lint rules, templates, and review checklists.

~~~ts
// Example feature-first layout
src/
  app/
  features/
    billing/
      api/
      components/
      hooks/
      types/
    dashboard/
  shared/
    ui/
    utils/
    lib/
~~~

**Key implementation point:** Architecture is about controlling change cost, not only code style.

---

## Rapid-Fire Advanced Follow-Up Prompts

- Explain how React scheduling prioritizes updates under load.
- Compare controlled vs uncontrolled inputs for high-frequency forms.
- Discuss hydration mismatch causes and prevention strategy.
- Show when to prefer key-based remount over effect-driven reset.
- Explain how to profile a render bottleneck with React DevTools Profiler.

## How To Use This For Interview Prep

1. Pick 3 questions/day and answer aloud in under 3 minutes each.
2. For each, implement one mini-demo and one failure case.
3. Add a tradeoff statement: performance, complexity, DX, testability.
4. Practice whiteboard architecture for one end-to-end React feature.
