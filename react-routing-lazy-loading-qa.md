# React Routing and Lazy Loading Interview Q&A

This guide is focused on interview questions for React Router and lazy loading.
It is suitable for intermediate to senior frontend interviews.

---

## 1) What is client-side routing in React, and why is it used?

**Short answer:**
Client-side routing lets the browser change views without full page reloads, creating a faster SPA experience.

**Explanation:**
In traditional apps, each navigation requests a new HTML page from the server. In React SPAs, the app loads once and React Router maps URL paths to components. This improves responsiveness and preserves client state between route changes.

**Code snippet:**

```tsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function Home() {
  return <h2>Home</h2>;
}

function About() {
  return <h2>About</h2>;
}

export default function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link> | <Link to="/about">About</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 2) What is the difference between BrowserRouter and HashRouter?

**Short answer:**
- BrowserRouter uses clean URLs like `/dashboard` and needs server fallback config.
- HashRouter uses URLs like `/#/dashboard` and works without server rewrite.

**Explanation:**
BrowserRouter depends on server support to return `index.html` for unknown routes. Without this, direct refresh on nested routes can produce 404. HashRouter avoids this by keeping route state after `#`, which is not sent to the server.

**Code snippet:**

```tsx
import { BrowserRouter, HashRouter } from 'react-router-dom';

// Preferred for production apps with proper server config
function BrowserApp() {
  return <BrowserRouter>{/* routes */}</BrowserRouter>;
}

// Useful in static hosting environments without rewrite support
function HashApp() {
  return <HashRouter>{/* routes */}</HashRouter>;
}
```

---

## 3) How do nested routes work in React Router v6?

**Short answer:**
Nested routes let you render parent layout and child route content together using `Outlet`.

**Explanation:**
Use nested `Route` definitions and place `<Outlet />` in the layout component where child pages should render.

**Code snippet:**

```tsx
import { BrowserRouter, Routes, Route, Outlet, Link } from 'react-router-dom';

function DashboardLayout() {
  return (
    <div>
      <h1>Dashboard</h1>
      <nav>
        <Link to="overview">Overview</Link> | <Link to="reports">Reports</Link>
      </nav>
      <Outlet />
    </div>
  );
}

function Overview() {
  return <p>Overview page</p>;
}

function Reports() {
  return <p>Reports page</p>;
}

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/dashboard" element={<DashboardLayout />}>
          <Route path="overview" element={<Overview />} />
          <Route path="reports" element={<Reports />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 4) How do you protect private routes?

**Short answer:**
Wrap protected pages with a guard component that checks authentication state and redirects with `Navigate` if unauthorized.

**Code snippet:**

```tsx
import { Navigate } from 'react-router-dom';

function RequireAuth({ isAuthenticated, children }: {
  isAuthenticated: boolean;
  children: JSX.Element;
}) {
  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }
  return children;
}

// Usage inside Routes
// <Route
//   path="/settings"
//   element={
//     <RequireAuth isAuthenticated={!!token}>
//       <Settings />
//     </RequireAuth>
//   }
// />
```

---

## 5) What are route params and query params, and how do you read them?

**Short answer:**
- Route params: part of the path, like `/users/:id`
- Query params: key-value in URL, like `/users?id=10&tab=activity`

**Code snippet:**

```tsx
import { useParams, useSearchParams } from 'react-router-dom';

function UserProfile() {
  const { id } = useParams();
  const [searchParams] = useSearchParams();
  const tab = searchParams.get('tab') ?? 'overview';

  return (
    <div>
      <h2>User ID: {id}</h2>
      <p>Tab: {tab}</p>
    </div>
  );
}
```

---

## 6) What is lazy loading in React?

**Short answer:**
Lazy loading delays loading of component code until needed, reducing initial bundle size and speeding first paint.

**Explanation:**
Use `React.lazy` with dynamic import and wrap lazy components with `Suspense` fallback.

**Code snippet:**

```tsx
import React, { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const HomePage = lazy(() => import('./pages/HomePage'));
const AnalyticsPage = lazy(() => import('./pages/AnalyticsPage'));

function Loader() {
  return <p>Loading page...</p>;
}

export default function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Loader />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/analytics" element={<AnalyticsPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

---

## 7) How do you apply lazy loading at route level?

**Short answer:**
Lazy-load each route component so only the active route chunk is downloaded.

**Explanation:**
Route-level lazy loading is one of the highest-impact frontend performance optimizations in large apps.

**Code snippet:**

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./features/dashboard/Dashboard'));
const Users = lazy(() => import('./features/users/Users'));
const Settings = lazy(() => import('./features/settings/Settings'));

export function AppRoutes() {
  return (
    <Suspense fallback={<div>Loading section...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/users" element={<Users />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

---

## 8) What are common mistakes with lazy loading and Suspense?

**Short answer:**
- Not wrapping lazy components with `Suspense`
- Putting one global boundary for everything
- Poor fallback UX
- Ignoring error handling for chunk-load failures

**Code snippet (better pattern):**

```tsx
import { lazy, Suspense } from 'react';

const Reports = lazy(() => import('./Reports'));

function ReportsSection() {
  return (
    <section>
      <h2>Reports</h2>
      {/* Local boundary keeps loading state scoped */}
      <Suspense fallback={<p>Loading reports...</p>}>
        <Reports />
      </Suspense>
    </section>
  );
}
```

---

## 9) How do you handle lazy-load failures (chunk load errors)?

**Short answer:**
Use an Error Boundary and optionally retry or refresh when dynamic import fails.

**Code snippet:**

```tsx
import React, { lazy, Suspense } from 'react';

const BillingPage = lazy(() => import('./BillingPage'));

class RouteErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <p>Failed to load this section.</p>
          <button onClick={() => window.location.reload()}>Retry</button>
        </div>
      );
    }

    return this.props.children;
  }
}

export function BillingRoute() {
  return (
    <RouteErrorBoundary>
      <Suspense fallback={<p>Loading billing...</p>}>
        <BillingPage />
      </Suspense>
    </RouteErrorBoundary>
  );
}
```

---

## 10) What is prefetching and when should you use it?

**Short answer:**
Prefetching loads likely-next route chunks in the background to improve perceived navigation speed.

**Code snippet:**

```tsx
import { Link } from 'react-router-dom';

// Simple manual prefetch on hover
function PrefetchUsersLink() {
  const prefetchUsers = () => {
    import('./features/users/Users');
  };

  return (
    <Link to="/users" onMouseEnter={prefetchUsers}>
      Users
    </Link>
  );
}
```

---

## 11) How do data loaders in React Router improve routing patterns?

**Short answer:**
Data routers fetch data before rendering route components, reducing loading-state boilerplate and centralizing route data logic.

**Code snippet (React Router data APIs):**

```tsx
import {
  createBrowserRouter,
  RouterProvider,
  useLoaderData,
} from 'react-router-dom';

async function usersLoader() {
  const res = await fetch('https://jsonplaceholder.typicode.com/users');
  if (!res.ok) throw new Error('Failed to fetch users');
  return res.json();
}

function UsersPage() {
  const users = useLoaderData() as Array<{ id: number; name: string }>;
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}

const router = createBrowserRouter([
  {
    path: '/users',
    loader: usersLoader,
    element: <UsersPage />,
  },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

---

## 12) How do you explain routing + lazy loading architecture in interviews?

**Strong interview answer format:**
1. Start with route hierarchy and layout routes.
2. Apply route-level lazy loading for feature pages.
3. Use local `Suspense` boundaries for better UX.
4. Add route guards for authentication and authorization.
5. Add error boundaries for failed chunk loads.
6. Prefetch likely next routes for smoother navigation.

**Sample explanation:**
"I design route trees with layout-based nesting, then lazy-load feature routes to keep initial bundle small. I avoid one giant Suspense boundary and place local boundaries by route section. I combine route guards and error boundaries for resilient navigation, and prefetch high-probability routes on hover or idle for speed."

---

## Rapid Fire Questions

1. Why does BrowserRouter need server rewrite config?
2. Difference between `Link` and `NavLink`?
3. When do you use nested routes with `Outlet`?
4. Why can one global Suspense hurt UX?
5. How do you handle 404 routes in React Router v6?
6. How do you pass state during navigation?
7. What causes chunk load errors in production?
8. Route params vs query params: when to choose each?

---

## Practice Tasks

1. Build a 3-level nested dashboard route structure using `Outlet`.
2. Convert all route components to lazy imports and add scoped `Suspense` fallbacks.
3. Add a protected route wrapper for `/admin`.
4. Add a wildcard `*` route for 404 page handling.
5. Add hover prefetch for two high-traffic routes.
