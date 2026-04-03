# React Intermediate Interview Q&A

This guide is designed for developers with around 2 to 5 years of React experience.
Each question includes:
- What interviewers expect
- A practical explanation
- A code snippet you can discuss

## 1) What is the difference between state and props in React?

**Short answer:**
- `props` are read-only inputs passed from parent to child.
- `state` is internal data managed by the component.

**Why interviewers ask:**
They want to confirm you understand one-way data flow and component responsibilities.

**Example:**

```tsx
import { useState } from 'react';

type CounterProps = {
  title: string; // prop from parent
};

function Counter({ title }: CounterProps) {
  const [count, setCount] = useState(0); // local state

  return (
    <div>
      <h3>{title}</h3>
      <p>Count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </div>
  );
}
```

**Interview tip:**
If data must be shared between siblings, lift state up to their common parent.

---

## 2) When should you use `useEffect`, and what are common mistakes?

**Short answer:**
Use `useEffect` for side effects, such as API calls, subscriptions, and DOM interactions.

**Common mistakes:**
- Missing dependencies
- Adding unnecessary dependencies
- Using effect for derived values that can be computed during render

**Example (fetch data once on mount):**

```tsx
import { useEffect, useState } from 'react';

type User = { id: number; name: string };

function UsersList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let active = true;

    async function loadUsers() {
      try {
        const res = await fetch('https://jsonplaceholder.typicode.com/users');
        const data = (await res.json()) as User[];
        if (active) setUsers(data);
      } finally {
        if (active) setLoading(false);
      }
    }

    loadUsers();

    return () => {
      active = false; // prevents updating state after unmount
    };
  }, []);

  if (loading) return <p>Loading...</p>;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

**Interview tip:**
Say clearly: "Effects synchronize with external systems."

---

## 3) What is the virtual DOM, and how does reconciliation work?

**Short answer:**
React creates an in-memory representation (virtual DOM), compares the new tree with the old tree, and updates only changed parts in the real DOM.

**Why it matters:**
This improves performance by minimizing expensive real DOM operations.

**Example discussion point:**
If a list item changes, React does not re-render the entire page. It updates only the affected nodes.

**Code snippet (list update):**

```tsx
import { useState } from 'react';

function TodoDemo() {
  const [items, setItems] = useState(['Learn React', 'Build project']);

  function addItem() {
    setItems((prev) => [...prev, `Task ${prev.length + 1}`]);
  }

  return (
    <div>
      <button onClick={addItem}>Add task</button>
      <ul>
        {items.map((item) => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    </div>
  );
}
```

**Interview tip:**
Mention that keys help React reconcile list items correctly.

---

## 4) Why are keys important in lists?

**Short answer:**
Keys provide stable identity for list items, helping React track which items changed, moved, or were removed.

**Bad key:** array index (especially if list order can change)
**Good key:** unique ID from data

```tsx
type Product = { id: string; name: string };

function ProductList({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

**Interview tip:**
Explain that wrong keys can cause subtle UI bugs, like wrong input values sticking to the wrong row.

---

## 5) What is prop drilling, and how can you avoid it?

**Short answer:**
Prop drilling is passing props through many intermediate components that do not use them.

**Solutions:**
- Context API
- State management libraries (Redux, Zustand, etc.)
- Component composition

**Example with Context API:**

```tsx
import { createContext, useContext } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');

function Toolbar() {
  return <ThemeButton />;
}

function ThemeButton() {
  const theme = useContext(ThemeContext);
  return <button>Theme: {theme}</button>;
}

export function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />
    </ThemeContext.Provider>
  );
}
```

**Interview tip:**
Mention that Context is good for low-to-medium frequency shared state, not always for all app state.

---

## 6) Explain controlled vs uncontrolled components.

**Controlled component:**
Input value is managed by React state.

**Uncontrolled component:**
Input state is managed by the DOM (often accessed via `ref`).

```tsx
import { useRef, useState } from 'react';

export function FormExamples() {
  const [name, setName] = useState(''); // controlled
  const emailRef = useRef<HTMLInputElement>(null); // uncontrolled

  function handleSubmit() {
    alert(`Name: ${name}, Email: ${emailRef.current?.value ?? ''}`);
  }

  return (
    <div>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Controlled name"
      />
      <input ref={emailRef} placeholder="Uncontrolled email" />
      <button onClick={handleSubmit}>Submit</button>
    </div>
  );
}
```

**Interview tip:**
Controlled inputs are preferred when you need validation and dynamic UI logic.

---

## 7) What is `React.memo`, and when should you use it?

**Short answer:**
`React.memo` prevents re-rendering a component when its props have not changed.

**Use it when:**
- Child renders are expensive
- Parent updates frequently
- Props are stable

```tsx
import React, { useState } from 'react';

const Child = React.memo(function Child({ label }: { label: string }) {
  console.log('Child rendered');
  return <p>{label}</p>;
});

export function MemoExample() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <Child label="I do not need to re-render each click" />
    </div>
  );
}
```

**Interview tip:**
Do not use memoization everywhere blindly. Measure first.

---

## 8) What is the difference between `useMemo` and `useCallback`?

**Short answer:**
- `useMemo` memoizes a computed value.
- `useCallback` memoizes a function reference.

```tsx
import { useCallback, useMemo, useState } from 'react';

export function MemoHooksExample({ numbers }: { numbers: number[] }) {
  const [multiplier, setMultiplier] = useState(2);

  const total = useMemo(() => {
    return numbers.reduce((sum, n) => sum + n * multiplier, 0);
  }, [numbers, multiplier]);

  const logTotal = useCallback(() => {
    console.log('Total:', total);
  }, [total]);

  return (
    <div>
      <p>Total: {total}</p>
      <button onClick={() => setMultiplier((m) => m + 1)}>Increase multiplier</button>
      <button onClick={logTotal}>Log total</button>
    </div>
  );
}
```

**Interview tip:**
Mention that both have overhead; use them where they solve real re-render or recalculation issues.

---

## 9) How do you handle API errors and loading states in React?

**Short answer:**
Track at least three states: loading, success, and error.

```tsx
import { useEffect, useState } from 'react';

type Post = { id: number; title: string };

export function Posts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function load() {
      try {
        const res = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=5');
        if (!res.ok) throw new Error('Failed to fetch posts');
        const data = (await res.json()) as Post[];
        setPosts(data);
      } catch (e) {
        setError(e instanceof Error ? e.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    }

    load();
  }, []);

  if (loading) return <p>Loading posts...</p>;
  if (error) return <p role="alert">Error: {error}</p>;

  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

**Interview tip:**
Explain retry behavior and user feedback for real-world production apps.

---

## 10) What are custom hooks, and why use them?

**Short answer:**
Custom hooks extract reusable stateful logic so components stay clean and focused.

```tsx
import { useEffect, useState } from 'react';

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() {
      setWidth(window.innerWidth);
    }

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}

export function ResponsiveLabel() {
  const width = useWindowWidth();
  return <p>Current width: {width}px</p>;
}
```

**Interview tip:**
Good custom hooks are small, reusable, and easy to test.

---

## 11) How do you optimize large lists in React?

**Short answer:**
Use list virtualization so only visible rows are rendered.

**Typical library:** `react-window`

```tsx
import { FixedSizeList as List, ListChildComponentProps } from 'react-window';

const items = Array.from({ length: 10000 }, (_, i) => `Row ${i + 1}`);

function Row({ index, style }: ListChildComponentProps) {
  return <div style={style}>{items[index]}</div>;
}

export function VirtualizedListExample() {
  return (
    <List
      height={300}
      width={300}
      itemCount={items.length}
      itemSize={35}
    >
      {Row}
    </List>
  );
}
```

**Interview tip:**
Talk about tradeoffs: virtualization improves performance but can complicate dynamic row heights and accessibility.

---

## 12) How would you explain lifting state up with an example?

**Short answer:**
Move shared state to the nearest common parent so multiple child components stay in sync.

```tsx
import { useState } from 'react';

function TemperatureInput({
  label,
  value,
  onChange,
}: {
  label: string;
  value: string;
  onChange: (v: string) => void;
}) {
  return (
    <label>
      {label}
      <input value={value} onChange={(e) => onChange(e.target.value)} />
    </label>
  );
}

export function TemperatureConverter() {
  const [celsius, setCelsius] = useState('');

  const fahrenheit = celsius === ''
    ? ''
    : ((Number(celsius) * 9) / 5 + 32).toFixed(1);

  return (
    <div>
      <TemperatureInput label="Celsius: " value={celsius} onChange={setCelsius} />
      <p>Fahrenheit: {fahrenheit}</p>
    </div>
  );
}
```

**Interview tip:**
Use this pattern before introducing global state libraries.

---

## 13) What is the difference between `useRef` and `useState`?

**Short answer:**
- `useState` updates trigger re-renders.
- `useRef` updates do not trigger re-renders.

```tsx
import { useRef, useState } from 'react';

export function RefVsState() {
  const [count, setCount] = useState(0);
  const renderCount = useRef(0);

  renderCount.current += 1;

  return (
    <div>
      <p>State count: {count}</p>
      <p>Render count (ref): {renderCount.current}</p>
      <button onClick={() => setCount((c) => c + 1)}>Increment state</button>
    </div>
  );
}
```

**Interview tip:**
Refs are ideal for mutable values, DOM access, and interval IDs.

---

## 14) How do you prevent memory leaks in React components?

**Short answer:**
Always clean up subscriptions, timers, and listeners in effect cleanup.

```tsx
import { useEffect, useState } from 'react';

export function Clock() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    const id = setInterval(() => {
      setTime(new Date());
    }, 1000);

    return () => clearInterval(id); // cleanup on unmount
  }, []);

  return <p>{time.toLocaleTimeString()}</p>;
}
```

**Interview tip:**
Mention cleanup as a key production reliability habit.

---

## 15) How should you structure a medium-size React project?

**Recommended feature-first structure:**

```txt
src/
  app/
  features/
    auth/
      components/
      hooks/
      services/
      types/
    products/
      components/
      hooks/
      services/
      types/
  shared/
    components/
    hooks/
    utils/
```

**Explanation:**
Feature-based folders reduce coupling, improve ownership, and scale better than purely technical folders.

---

## Rapid Fire Intermediate Questions

1. What causes unnecessary re-renders and how do you debug them?
2. Why should side effects not run in render?
3. What is the purpose of dependency arrays in hooks?
4. How do you handle form validation in React?
5. When would you choose Context over Redux?

---

## How to Practice With This File

1. Pick 5 questions and answer them in under 2 minutes each.
2. For each answer, include one real project example.
3. Implement at least 3 snippets in a sandbox and explain tradeoffs.
4. Practice follow-up questions like performance, scalability, and testing.
