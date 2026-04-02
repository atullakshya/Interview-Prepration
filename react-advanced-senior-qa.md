# Advanced React Interview Q&A (6+ Years Experience)

This guide is for experienced developers with 6+ years of React, Redux, authentication, and modern tooling experience.
Each question includes deep explanations, production-ready code, and architectural insights.

---

## REACT ADVANCED

### 1) How do you optimize performance in a large-scale React application?

**Short answer:**
Use code splitting, lazy loading, memoization, virtualization, and React.lazy for route-based splitting. Monitor with DevTools Profiler.

**Deep explanation:**
At scale, you need multiple strategies: reduce bundle size, defer non-critical rendering, prevent unnecessary re-renders, and optimize network requests.

**Code example:**

```tsx
import React, { Suspense, lazy, useMemo, useCallback } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// 1. Route-based code splitting with React.lazy
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Settings = lazy(() => import('./pages/Settings'));

// 2. Component-level splitting for large lists
const HeavyList = lazy(() => import('./components/HeavyList'));

// Loading fallback
function LoadingSpinner() {
  return <div className="spinner">Loading...</div>;
}

// 3. Memoized expensive computation
export function ExpensiveComponent({ data, searchTerm }) {
  const filtered = useMemo(() => {
    console.log('Computing filtered results...');
    return data.filter((item) =>
      item.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [data, searchTerm]); // Only recompute if data or searchTerm changes

  return (
    <ul>
      {filtered.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}

// 4. Callback memoization for child props
function Parent({ onFilter }) {
  const handleFilter = useCallback((term) => {
    onFilter(term);
  }, []); // Stable callback identity

  return <Child onFilter={handleFilter} />;
}

const Child = React.memo(function Child({ onFilter }) {
  // Won't re-render unless onFilter changes (it won't)
  return <input onChange={(e) => onFilter(e.target.value)} />;
});

// 5. Virtualized list for 10k+ items
import { FixedSizeList } from 'react-window';

function VirtualizedHugeList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}

// 6. Route-level lazy loading
export function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/analytics" element={<Analytics />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// 7. Image lazy loading with Intersection Observer
function LazyImage({ src, alt }) {
  const ref = React.useRef(null);
  const [imageSrc, setImageSrc] = React.useState(null);

  React.useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setImageSrc(src); // Load image when visible
          observer.unobserve(entry.target);
        }
      },
      { threshold: 0.1 }
    );

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [src]);

  return <img ref={ref} src={imageSrc} alt={alt} />;
}
```

**Interview tip:**
"At scale, profiling is critical. Use React DevTools Profiler to identify actual bottlenecks, not perceived ones."

---

### 2) Explain concurrent rendering and Suspense boundaries in React 18+

**Short answer:**
Concurrent rendering lets React interrupt work to handle high-priority updates. Suspense lets you declare async dependencies cleanly.

**Deep explanation:**
React 18 introduced useTransition and useDeferredValue for concurrent updates. Suspense integrates with data fetching frameworks.

**Code example:**

```tsx
import { Suspense, useTransition, useDeferredValue } from 'react';

// 1. useTransition: Mark update as non-blocking
function SearchUsers({ users }) {
  const [query, setQuery] = React.useState('');
  const [isPending, startTransition] = useTransition();

  const handleSearch = (e) => {
    const value = e.target.value;
    setQuery(value); // Urgent: keeps input responsive

    // Non-urgent: can be interrupted
    startTransition(() => {
      // Heavy filter operation
      const filtered = users.filter((u) =>
        u.name.toLowerCase().includes(value.toLowerCase())
      );
      // Update filtered results
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={handleSearch}
        placeholder="Search..."
      />
      {isPending && <p>Searching...</p>}
      {/* Results rendered here */}
    </div>
  );
}

// 2. useDeferredValue: Defer consuming a value
function DeferredValueExample({ searchQuery }) {
  const deferredQuery = useDeferredValue(searchQuery);
  const isStale = searchQuery !== deferredQuery;

  return (
    <div>
      {isStale && <p>Updating results...</p>}
      <SearchResults query={deferredQuery} />
    </div>
  );
}

// 3. Suspense + Server Components / Data Fetching Library
// (This is the future pattern with frameworks like Next.js 13+)
function UserProfileAsync({ userId }) {
  return (
    <Suspense fallback={<div>Loading profile...</div>}>
      <UserProfile userId={userId} />
    </Suspense>
  );
}

// Simulated async component (in real app, use framework)
async function UserProfile({ userId }) {
  const user = await fetch(`/api/users/${userId}`).then((r) => r.json());
  return <div>{user.name}</div>;
}

// 4. Multiple Suspense boundaries for granular control
export function ComplexPage() {
  return (
    <div>
      {/* Each boundary can show independently */}
      <Suspense fallback={<div>Loading header...</div>}>
        <Header />
      </Suspense>

      <Suspense fallback={<div>Loading content...</div>}>
        <MainContent />
      </Suspense>

      <Suspense fallback={<div>Loading sidebar...</div>}>
        <Sidebar />
      </Suspense>
    </div>
  );
}

// 5. useTransition with form submission
function FormWithTransition() {
  const [isPending, startTransition] = useTransition();

  const handleSubmit = (e) => {
    e.preventDefault();
    startTransition(async () => {
      const formData = new FormData(e.target);
      const result = await fetch('/api/submit', {
        method: 'POST',
        body: formData,
      });
      // Handle result
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

**Interview tip:**
"Concurrent features move React toward non-blocking rendering. useTransition marks work as low-priority; Suspense declares async boundaries."

---

### 3) How do you handle error boundaries and error recovery strategies?

**Short answer:**
Use Error Boundaries to catch render errors and provide fallbacks. Implement retry logic and error reporting for production resilience.

**Code example:**

```tsx
import React from 'react';

// 1. Comprehensive Error Boundary
type ErrorBoundaryState = {
  hasError: boolean;
  error: Error | null;
  errorCount: number;
};

export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  ErrorBoundaryState
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null, errorCount: 0 };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error, errorCount: 0 };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Log to external service (Sentry, DataDog, etc.)
    console.error('Error caught:', error, errorInfo);
    this.logErrorToService(error, errorInfo);
  }

  logErrorToService(error: Error, errorInfo: React.ErrorInfo) {
    // Example: integrate with Sentry
    if (typeof window !== 'undefined' && window.Sentry) {
      window.Sentry.captureException(error, { contexts: errorInfo });
    }
  }

  handleRetry = () => {
    this.setState((prev) => ({
      hasError: false,
      error: null,
      errorCount: prev.errorCount + 1,
    }));
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-container">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={this.handleRetry}>
            Try again (Attempt {this.state.errorCount + 1})
          </button>
          {this.state.errorCount > 2 && (
            <p>Multiple failures detected. Please refresh the page.</p>
          )}
        </div>
      );
    }

    return this.props.children;
  }
}

// 2. Granular error boundaries for features
export function FeatureWithErrorBoundary() {
  return (
    <ErrorBoundary>
      <HeavyFeature />
    </ErrorBoundary>
  );
}

// 3. Hook-based error handling (custom)
function useAsyncError() {
  const [, setError] = React.useState();

  React.useEffect(() => {
    if (window.__ASYNC_ERROR__) {
      setError(window.__ASYNC_ERROR__);
    }
  }, []);

  return (error: Error) => {
    setError(() => {
      throw error; // Throw to nearest Error Boundary
    });
  };
}

// 4. Async error handling in components
function AsyncComponent() {
  const throwError = useAsyncError();

  React.useEffect(() => {
    fetchData()
      .catch((error) => {
        throwError(error); // Caught by Error Boundary
      });
  }, [throwError]);

  return <div>Content</div>;
}

// 5. Resilient data fetching with retry logic
async function fetchWithRetry(
  url: string,
  options: RequestInit = {},
  maxRetries = 3
): Promise<Response> {
  let lastError: Error | null = null;

  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response;
    } catch (error) {
      lastError = error as Error;
      if (i < maxRetries - 1) {
        // Exponential backoff
        await new Promise((resolve) =>
          setTimeout(resolve, Math.pow(2, i) * 1000)
        );
      }
    }
  }

  throw lastError;
}
```

**Interview tip:**
"Error Boundaries catch render errors, not event handlers. For async errors, throw from setState or useEffect. Use a service like Sentry for production visibility."

---

### 4) Explain advanced Context patterns and when to avoid them

**Short answer:**
Context is good for low-frequency updates (theme, auth). For high-frequency updates, use external stores or split contexts by update frequency.

**Code example:**

```tsx
import React, { createContext, useContext, useCallback } from 'react';

// ❌ ANTI-PATTERN: Monolithic context with mixed update frequencies
const BadAuthContext = createContext<{
  user: User;
  isLoading: boolean;
  currentFormInput: string; // High frequency
  loginCount: number; // Low frequency
}>(null);

// ✅ PATTERN 1: Split contexts by update frequency
const UserContext = createContext<{ user: User }>(null);
const AuthLoadingContext = createContext<{ isLoading: boolean }>(null);

// Only slow-changing data
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = React.useState<User | null>(null);

  const value = React.useMemo(() => ({ user }), [user]); // Stable

  return (
    <UserContext.Provider value={value}>{children}</UserContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error('useAuth requires AuthProvider');
  return ctx;
}

// ✅ PATTERN 2: Callback-based state for derived values
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = React.useState<'light' | 'dark'>('light');

  const toggleTheme = useCallback(() => {
    setTheme((t) => (t === 'light' ? 'dark' : 'light'));
  }, []);

  // Memoize to prevent unnecessary renders in children
  const value = React.useMemo(
    () => ({ theme, toggleTheme }),
    [theme, toggleTheme]
  );

  return (
    <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
  );
}

// ✅ PATTERN 3: Custom hook for selecting specific context value
const NotificationContext = createContext<{
  notifications: Notification[];
  addNotification: (msg: string) => void;
}>(null);

export function useNotifications() {
  const ctx = useContext(NotificationContext);
  if (!ctx) throw new Error('useNotifications requires Provider');
  return ctx;
}

// Consumer can memoize selected value
function NotificationDisplay() {
  const { notifications } = useNotifications();

  return (
    <div>
      {notifications.map((n) => (
        <div key={n.id}>{n.message}</div>
      ))}
    </div>
  );
}

// ✅ PATTERN 4: Use external store for high-frequency state
// (Zustand, Jotai, Recoil, or Valtio)
import { create } from 'zustand';

const useFormStore = create((set) => ({
  formData: { name: '', email: '' },
  setFormData: (data) => set({ formData: data }),
}));

function HighFrequencyForm() {
  const { formData, setFormData } = useFormStore();

  return (
    <input
      value={formData.name}
      onChange={(e) =>
        setFormData({ ...formData, name: e.target.value })
      }
    />
  );
}

// ✅ PATTERN 5: Combining Context + External Store
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <NotificationProvider>{children}</NotificationProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```

**Interview tip:**
"Context is for infrequent global state. For frequently changing state, Zustand or Redux are better choices to prevent cascading renders."

---

### 5) What are Web Workers, and how do you offload heavy computation?

**Short answer:**
Web Workers run JavaScript in a separate thread without blocking the main thread. Use them for CPU-intensive tasks like image processing or data parsing.

**Code example:**

```tsx
// main.tsx - Main thread component
import React from 'react';

function ImageProcessor() {
  const [processed, setProcessed] = React.useState<Blob | null>(null);
  const [isProcessing, setIsProcessing] = React.useState(false);
  const workerRef = React.useRef<Worker | null>(null);

  React.useEffect(() => {
    // Initialize worker once
    workerRef.current = new Worker(
      new URL('./imageWorker.ts', import.meta.url),
      { type: 'module' }
    );

    workerRef.current.onmessage = (event: MessageEvent) => {
      const { processedImage } = event.data;
      setProcessed(processedImage);
      setIsProcessing(false);
    };

    return () => {
      workerRef.current?.terminate();
    };
  }, []);

  const handleImageUpload = async (file: File) => {
    setIsProcessing(true);
    const arrayBuffer = await file.arrayBuffer();

    // Send to worker (transferable for zero-copy)
    workerRef.current?.postMessage(
      { imageBuffer: arrayBuffer },
      [arrayBuffer] // Transfer ownership, don't copy
    );
  };

  return (
    <div>
      <input type="file" accept="image/*" onChange={(e) => {
        if (e.target.files?.[0]) {
          handleImageUpload(e.target.files[0]);
        }
      }} />
      {isProcessing && <p>Processing...</p>}
      {processed && (
        <img src={URL.createObjectURL(processed)} alt="Processed" />
      )}
    </div>
  );
}

export default ImageProcessor;
```

```typescript
// imageWorker.ts - Web Worker thread
self.onmessage = async (event: MessageEvent) => {
  const { imageBuffer } = event.data;

  // Heavy computation off main thread
  const processedImage = await processImage(imageBuffer);

  // Send result back
  self.postMessage({ processedImage }, [processedImage]);
};

async function processImage(buffer: ArrayBuffer): Promise<Blob> {
  // Simulate processing (in reality: image filters, compression, etc.)
  const data = new Uint8Array(buffer);
  // Apply filters, transformations, etc.
  return new Blob([data], { type: 'image/png' });
}
```

**Interview tip:**
"Use Web Workers for CPU-bound tasks (parsing, filtering, crypto). Use Transferable objects to avoid copying large ArrayBuffers."

---

## REDUX ADVANCED

### 6) What is Redux Saga, and how does it compare to Redux Thunk?

**Short answer:**
Redux Saga uses generator functions to manage side effects. More powerful than Thunk for complex async flows, but more boilerplate.

**Code example:**

```tsx
import { call, put, takeEvery, select } from 'redux-saga/effects';
import type { SagaIterator } from 'redux-saga';

// Slice
const postsSlice = createSlice({
  name: 'posts',
  initialState: { data: [], loading: false, error: null },
  reducers: {
    fetchStarted: (state) => {
      state.loading = true;
    },
    fetchSuccess: (state, action) => {
      state.data = action.payload;
      state.loading = false;
    },
    fetchError: (state, action) => {
      state.error = action.payload;
      state.loading = false;
    },
  },
});

// Selector
const selectUserId = (state) => state.auth.userId;

// API call
async function fetchPostsAPI(userId: number) {
  const res = await fetch(`/api/posts?userId=${userId}`);
  return res.json();
}

// SAGA: Generator-based side effect
function* fetchPostsSaga(action): SagaIterator<void> {
  try {
    // Select state
    const userId = yield select(selectUserId);

    // Call API
    const posts = yield call(fetchPostsAPI, userId);

    // Dispatch success
    yield put(postsSlice.actions.fetchSuccess(posts));
  } catch (error) {
    // Dispatch error
    yield put(postsSlice.actions.fetchError((error as Error).message));
  }
}

// Root saga
function* rootSaga(): SagaIterator<void> {
  // Listen for action
  yield takeEvery('posts/fetchRequested', fetchPostsSaga);
}

// Configure store with Saga middleware
import createSagaMiddleware from 'redux-saga';

const sagaMiddleware = createSagaMiddleware();

const store = configureStore({
  reducer: {
    posts: postsSlice.reducer,
  },
  middleware: (getDefault) =>
    getDefault().concat(sagaMiddleware),
});

sagaMiddleware.run(rootSaga);

// ✅ COMPARISON: Thunk vs Saga
// Thunk:
const fetchPostsThunk = () => async (dispatch) => {
  try {
    const posts = await fetchPostsAPI();
    dispatch(fetchSuccess(posts));
  } catch (e) {
    dispatch(fetchError(e.message));
  }
};

// Saga:
function* fetchPostsSaga() {
  try {
    const posts = yield call(fetchPostsAPI);
    yield put(fetchSuccess(posts));
  } catch (e) {
    yield put(fetchError(e.message));
  }
}

// Saga advantages:
// - Easier to test (generator, not async)
// - Better for complex flows (race, fork, join)
// - Centralized side effect logic

// Thunk advantages:
// - Simpler, less boilerplate
// - Easier to learn
// - Sufficient for most apps
```

**Interview tip:**
"Saga is overkill for simple apps. Use Thunk for most cases. Consider Saga only when you have complex side-effect orchestration (race conditions, cancellation, retry)."

---

### 7) How do you handle real-time data with Redux and WebSockets?

**Short answer:**
Create a middleware to connect WebSocket, dispatch actions on message, and clean up on disconnect. Consider normalized state for real-time updates.

**Code example:**

```tsx
// socketMiddleware.ts
import { AnyAction, Middleware } from '@reduxjs/toolkit';

export const socketMiddleware: Middleware = (store) => (next) => (action: AnyAction) => {
  // Allow action through first
  const result = next(action);

  // Connect WebSocket on login
  if (action.type === 'auth/loginSuccess') {
    const token = store.getState().auth.token;
    connectWebSocket(store, token);
  }

  // Disconnect on logout
  if (action.type === 'auth/logout') {
    disconnectWebSocket();
  }

  return result;
};

let ws: WebSocket | null = null;

function connectWebSocket(store, token: string) {
  ws = new WebSocket(`wss://api.example.com/ws?token=${token}`);

  ws.onmessage = (event) => {
    const message = JSON.parse(event.data);

    // Dispatch action based on message type
    switch (message.type) {
      case 'POST_CREATED':
        store.dispatch({
          type: 'posts/addPost',
          payload: message.data,
        });
        break;

      case 'POST_UPDATED':
        store.dispatch({
          type: 'posts/updatePost',
          payload: message.data,
        });
        break;

      case 'USER_ONLINE':
        store.dispatch({
          type: 'presence/setUserOnline',
          payload: message.userId,
        });
        break;

      default:
        console.warn('Unknown message type:', message.type);
    }
  };

  ws.onerror = (error) => {
    console.error('WebSocket error:', error);
    store.dispatch({ type: 'socket/error', payload: error });
  };

  ws.onclose = () => {
    // Attempt reconnection with exponential backoff
    scheduleReconnect(store);
  };
}

function disconnectWebSocket() {
  if (ws) {
    ws.close();
    ws = null;
  }
}

let reconnectAttempts = 0;

function scheduleReconnect(store) {
  const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
  setTimeout(() => {
    reconnectAttempts++;
    const token = store.getState().auth.token;
    connectWebSocket(store, token);
  }, delay);
}

// Slice for real-time data (normalized)
const postsSlice = createSlice({
  name: 'posts',
  initialState: { byId: {}, allIds: [] },
  reducers: {
    addPost: (state, action) => {
      const post = action.payload;
      state.byId[post.id] = post;
      state.allIds.push(post.id);
    },
    updatePost: (state, action) => {
      const { id, ...updates } = action.payload;
      if (state.byId[id]) {
        state.byId[id] = { ...state.byId[id], ...updates };
      }
    },
    removePost: (state, action) => {
      delete state.byId[action.payload];
      state.allIds = state.allIds.filter((id) => id !== action.payload);
    },
  },
});

// Selector
export const selectPost = (state, postId) => state.posts.byId[postId];
export const selectAllPosts = (state) =>
  state.posts.allIds.map((id) => state.posts.byId[id]);
```

**Interview tip:**
"WebSocket + Redux requires careful middleware design. Normalize state to make updates idempotent. Implement backoff for reconnection."

---

## MSAL (Microsoft Authentication Library)

### 8) How do you implement MSAL for Azure AD authentication in React?

**Short answer:**
MSAL handles OAuth 2.0 / OpenID Connect flow with Azure AD. Use `@azure/msal-react` with PublicClientApplication for SPA auth.

**Code example:**

```tsx
import {
  PublicClientApplication,
  BrowserCacheLocation,
  LogLevel,
} from '@azure/msal-browser';
import { MsalProvider, useIsAuthenticated, useMsal } from '@azure/msal-react';
import React from 'react';

// 1. Configure MSAL
const msalConfig = {
  auth: {
    clientId: process.env.REACT_APP_MSAL_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${process.env.REACT_APP_MSAL_TENANT_ID}`,
    redirectUri: window.location.origin, // Callback URL
    postLogoutRedirectUri: window.location.origin,
  },
  cache: {
    cacheLocation: BrowserCacheLocation.LocalStorage, // or SessionStorage
    storeAuthStateInCookie: true, // Recommended for IE11
  },
  system: {
    loggerOptions: {
      loggerCallback: (level, message, containsPii) => {
        if (containsPii) return;
        console.log(`[${level}] ${message}`);
      },
      logLevel: LogLevel.Verbose,
      piiLoggingEnabled: false,
    },
  },
};

const msalInstance = new PublicClientApplication(msalConfig);

// 2. Handle redirect after sign-in
msalInstance.handleRedirectPromise().catch((error) => {
  console.error('Error handling redirect:', error);
});

// 3. Root component with MsalProvider
export function App() {
  return (
    <MsalProvider instance={msalInstance}>
      <MainApp />
    </MsalProvider>
  );
}

// 4. Login component
function LoginButton() {
  const { instance } = useMsal();
  const isAuthenticated = useIsAuthenticated();

  const loginRequest = {
    scopes: ['user.read'], // Microsoft Graph scopes
  };

  const handleLogin = async () => {
    try {
      // Redirect to login
      await instance.loginPopup(loginRequest);
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  const handleLogout = async () => {
    await instance.logout({
      postLogoutRedirectUri: '/auth/logout-complete',
    });
  };

  return (
    <div>
      {isAuthenticated ? (
        <>
          <p>Signed in</p>
          <button onClick={handleLogout}>Sign out</button>
        </>
      ) : (
        <button onClick={handleLogin}>Sign in</button>
      )}
    </div>
  );
}

// 5. Get access token for API calls
function ProtectedComponent() {
  const { instance, accounts } = useMsal();
  const [accessToken, setAccessToken] = React.useState<string | null>(null);

  React.useEffect(() => {
    const acquireToken = async () => {
      const account = accounts[0];
      if (!account) return;

      try {
        const response = await instance.acquireTokenSilent({
          scopes: ['https://graph.microsoft.com/.default'],
          account,
        });
        setAccessToken(response.accessToken);
      } catch (error) {
        console.error('Token acquisition failed:', error);

        // Fallback to interactive
        const response = await instance.acquireTokenPopup({
          scopes: ['https://graph.microsoft.com/.default'],
        });
        setAccessToken(response.accessToken);
      }
    };

    acquireToken();
  }, [instance, accounts]);

  return <div>Token: {accessToken?.substring(0, 20)}...</div>;
}

// 6. Require authentication wrapper
export function RequireAuth({ children }: { children: React.ReactNode }) {
  const isAuthenticated = useIsAuthenticated();

  if (!isAuthenticated) {
    return <div>Please sign in to continue</div>;
  }

  return <>{children}</>;
}

// 7. Redux integration with MSAL
const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, tokens: {} },
  reducers: {
    setUser: (state, action) => {
      state.user = action.payload;
    },
    setTokens: (state, action) => {
      state.tokens = action.payload;
    },
  },
});

function AppWithRedux() {
  const { instance, accounts } = useMsal();
  const dispatch = useDispatch();

  React.useEffect(() => {
    if (accounts.length > 0) {
      const account = accounts[0];
      dispatch(authSlice.actions.setUser(account));
    }
  }, [accounts, dispatch]);

  return <MainApp />;
}
```

**Interview tip:**
"MSAL handles OAuth flow transparently. Key patterns: handle redirect after login, acquire tokens silently with fallback to interactive, use scopes for permissions."

---

### 9) How do you handle token refresh and expiration with MSAL?

**Short answer:**
MSAL auto-refreshes with `acquireTokenSilent`. Configure token cache, handle MsalInteractionRequiredError for re-authentication.

**Code example:**

```tsx
// tokenManager.ts
import { PublicClientApplication, InteractionRequiredError } from '@azure/msal-browser';

export class TokenManager {
  constructor(private msalInstance: PublicClientApplication) {}

  async getAccessToken(
    scopes: string[],
    account: any
  ): Promise<string> {
    try {
      // Try silent acquisition first
      const response = await this.msalInstance.acquireTokenSilent({
        scopes,
        account,
      });
      return response.accessToken;
    } catch (error) {
      if (error instanceof InteractionRequiredError) {
        // Token expired or unavailable, need user interaction
        console.log('Token expired, re-authenticating...');
        const response = await this.msalInstance.acquireTokenPopup({
          scopes,
        });
        return response.accessToken;
      }
      throw error;
    }
  }

  // Intercept HTTP requests to add token
  createAuthenticationHeader(token: string): Record<string, string> {
    return {
      Authorization: `Bearer ${token}`,
    };
  }
}

// Hook for component use
function useAccessToken(scopes: string[]) {
  const { instance, accounts } = useMsal();
  const [token, setToken] = React.useState<string | null>(null);
  const [isLoading, setIsLoading] = React.useState(false);
  const [error, setError] = React.useState<Error | null>(null);

  React.useEffect(() => {
    const getToken = async () => {
      if (!accounts.length) return;

      setIsLoading(true);
      try {
        const response = await instance.acquireTokenSilent({
          scopes,
          account: accounts[0],
        });
        setToken(response.accessToken);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    };

    getToken();

    // Refresh token before expiration
    const intervalId = setInterval(() => {
      getToken();
    }, 5 * 60 * 1000); // Every 5 minutes

    return () => clearInterval(intervalId);
  }, [instance, accounts, scopes]);

  return { token, isLoading, error };
}

// Usage
function ApiComponent() {
  const { token } = useAccessToken(['https://graph.microsoft.com/.default']);

  React.useEffect(() => {
    if (token) {
      fetch('/api/data', {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }).then((r) => r.json());
    }
  }, [token]);

  return <div>Loading data...</div>;
}
```

**Interview tip:**
"MSAL handles token refresh automatically in `acquireTokenSilent`. Cache is encrypted in localStorage by default. Handle InteractionRequiredError for expired scenarios."

---

## AXIOS

### 10) How do you create a robust Axios instance with interceptors for authentication and error handling?

**Short answer:**
Create an Axios instance, add request/response interceptors for auth headers and error handling, implement retry logic.

**Code example:**

```tsx
import axios, { AxiosInstance, AxiosError, AxiosResponse } from 'axios';

// 1. Create Axios instance
const axiosInstance: AxiosInstance = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 2. Request interceptor: Add auth token
axiosInstance.interceptors.request.use(
  (config) => {
    // Get token from Redux, Context, or localStorage
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 3. Response interceptor: Handle errors and token refresh
axiosInstance.interceptors.response.use(
  (response: AxiosResponse) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as any;

    // Handle 401 Unauthorized
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Attempt to refresh token
        const refreshToken = localStorage.getItem('refreshToken');
        const response = await axios.post(
          `${process.env.REACT_APP_API_URL}/auth/refresh`,
          { refreshToken }
        );

        const { accessToken } = response.data;
        localStorage.setItem('accessToken', accessToken);

        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return axiosInstance(originalRequest);
      } catch (refreshError) {
        // Refresh failed, logout user
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    // Handle other errors
    if (error.response?.status === 403) {
      console.error('Forbidden: Access denied');
    } else if (error.response?.status === 404) {
      console.error('Not found:', error.config?.url);
    } else if (error.response?.status >= 500) {
      console.error('Server error:', error.response.status);
    }

    return Promise.reject(error);
  }
);

// 4. Retry utility for specific errors
const retryAxios = axiosInstance.defaults;

interface RetryConfig extends AxiosError {
  retryCount?: number;
  maxRetries?: number;
}

axiosInstance.interceptors.response.use(undefined, async (error: RetryConfig) => {
  const { maxRetries = 3, retryCount = 0, config } = error;

  // Retry on network error or specific status codes
  if (
    retryCount < maxRetries &&
    (!error.response || [408, 429, 500, 502, 503].includes(error.response.status))
  ) {
    const delay = Math.pow(2, retryCount) * 1000; // Exponential backoff
    await new Promise((resolve) => setTimeout(resolve, delay));

    const newConfig = { ...config, retryCount: retryCount + 1 };
    return axiosInstance(newConfig);
  }

  return Promise.reject(error);
});

// 5. API service layer
export class ApiService {
  static async getUser(id: string) {
    return axiosInstance.get(`/users/${id}`);
  }

  static async updateUser(id: string, data: any) {
    return axiosInstance.put(`/users/${id}`, data);
  }

  static async createPost(data: any) {
    return axiosInstance.post('/posts', data);
  }

  static async deletePost(id: string) {
    return axiosInstance.delete(`/posts/${id}`);
  }
}

// 6. Use in React component
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = React.useState(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState<string | null>(null);

  React.useEffect(() => {
    ApiService.getUser(userId)
      .then((response) => setUser(response.data))
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{JSON.stringify(user)}</div>;
}

export { axiosInstance, ApiService };
```

**Interview tip:**
"Interceptors are key for auth token injection and error handling. Use exponential backoff for retries. Separate API calls into a service layer for reusability."

---

### 11) How do you handle file uploads with Axios and show progress?

**Short answer:**
Use FormData, set `Content-Type: multipart/form-data`, track progress with onUploadProgress.

**Code example:**

```tsx
function FileUpload() {
  const [progress, setProgress] = React.useState(0);
  const [file, setFile] = React.useState<File | null>(null);

  const handleUpload = async (selectedFile: File) => {
    const formData = new FormData();
    formData.append('file', selectedFile);

    try {
      const response = await axiosInstance.post('/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
        onUploadProgress: (progressEvent) => {
          if (progressEvent.total) {
            const percentComplete = (progressEvent.loaded / progressEvent.total) * 100;
            setProgress(percentComplete);
          }
        },
      });

      console.log('Upload successful:', response.data);
    } catch (error) {
      console.error('Upload failed:', error);
    }
  };

  return (
    <div>
      <input
        type="file"
        onChange={(e) => {
          if (e.target.files?.[0]) {
            setFile(e.target.files[0]);
            handleUpload(e.target.files[0]);
          }
        }}
      />
      {progress > 0 && (
        <div className="progress-bar">
          <div style={{ width: `${progress}%` }}>{Math.round(progress)}%</div>
        </div>
      )}
    </div>
  );
}
```

---

## CSS ADVANCED

### 12) How do you create a scalable CSS architecture for large applications?

**Short answer:**
Use CSS-in-JS, CSS Modules, or BEM + SCSS. Structure by component, maintain naming conventions, avoid global namespace pollution.

**Code example:**

```tsx
// ✅ PATTERN 1: CSS Modules (Recommended for MPA/larger apps)
// Button.module.scss
.button {
  padding: 10px 16px;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s ease;
  border: none;

  &:hover {
    opacity: 0.8;
  }

  &:active {
    transform: scale(0.98);
  }

  &.primary {
    background: #007bff;
    color: white;

    &:hover {
      background: #0056b3;
    }
  }

  &.secondary {
    background: #6c757d;
    color: white;
  }

  &.disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}

// Button.tsx
import styles from './Button.module.scss';

interface ButtonProps {
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
}

export function Button({
  variant = 'primary',
  disabled = false,
  children,
  onClick,
}: ButtonProps) {
  return (
    <button
      className={`${styles.button} ${styles[variant]} ${
        disabled ? styles.disabled : ''
      }`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}

// ✅ PATTERN 2: CSS-in-JS with Emotion (Best for dynamic theming)
import { css } from '@emotion/react';
import styled from '@emotion/styled';

const buttonStyles = css`
  padding: 10px 16px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;

  &:hover {
    opacity: 0.8;
  }
`;

interface StyledButtonProps {
  variant: 'primary' | 'secondary';
}

const StyledButton = styled.button<StyledButtonProps>`
  ${buttonStyles}
  background: ${(props) => (props.variant === 'primary' ? '#007bff' : '#6c757d')};
  color: white;

  &:hover {
    background: ${(props) =>
      props.variant === 'primary' ? '#0056b3' : '#5a6268'};
  }
`;

// ✅ PATTERN 3: Tailwind (Utility-first, great for rapid development)
export function TailwindButton({
  variant = 'primary',
  children,
}: ButtonProps) {
  const baseStyles = 'px-4 py-2 rounded transition-all duration-200';
  const variantStyles =
    variant === 'primary'
      ? 'bg-blue-600 text-white hover:bg-blue-700'
      : 'bg-gray-600 text-white hover:bg-gray-700';

  return <button className={`${baseStyles} ${variantStyles}`}>{children}</button>;
}

// ✅ PATTERN 4: BEM + SCSS for component-based structure
// components/Card/Card.scss
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
  background: white;

  &__header {
    font-size: 18px;
    font-weight: bold;
    margin-bottom: 12px;
  }

  &__content {
    font-size: 14px;
    line-height: 1.6;
  }

  &__footer {
    margin-top: 16px;
    padding-top: 16px;
    border-top: 1px solid #eee;
  }

  &--elevated {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  &--loading {
    opacity: 0.6;
    pointer-events: none;
  }
}

// components/Card/Card.tsx
import './Card.scss';

interface CardProps {
  title: string;
  children: React.ReactNode;
  elevated?: boolean;
  loading?: boolean;
}

export function Card({ title, children, elevated, loading }: CardProps) {
  const classes = [
    'card',
    elevated && 'card--elevated',
    loading && 'card--loading',
  ]
    .filter(Boolean)
    .join(' ');

  return (
    <div className={classes}>
      <h2 className="card__header">{title}</h2>
      <div className="card__content">{children}</div>
    </div>
  );
}

// ✅ PATTERN 5: Theme management with CSS Variables + Context
// theme.ts
export const lightTheme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#212529',
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
  },
};

export const darkTheme = {
  colors: {
    primary: '#0d6efd',
    secondary: '#6c757d',
    background: '#121212',
    text: '#e0e0e0',
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
  },
};

// ThemeProvider.tsx
const ThemeContext = React.createContext(lightTheme);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = React.useState(lightTheme);

  React.useEffect(() => {
    // Set CSS variables
    const root = document.documentElement;
    Object.entries(theme.colors).forEach(([key, value]) => {
      root.style.setProperty(`--color-${key}`, value);
    });
  }, [theme]);

  return (
    <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
  );
}

// styles/global.css
:root {
  --color-primary: #007bff;
  --color-secondary: #6c757d;
  --color-background: #ffffff;
  --color-text: #212529;
}

body {
  background: var(--color-background);
  color: var(--color-text);
}

.button {
  background: var(--color-primary);
  color: white;
}
```

**Interview tip:**
"CSS Modules prevent name collisions. CSS-in-JS enables dynamic theming. Tailwind speeds up development. Choose based on project scale and team preference."

---

### 13) How do you implement responsive design and optimize for mobile?

**Short answer:**
Mobile-first approach, semantic breakpoints, flexible layouts, optimize images, minimize CSS/JS.

**Code example:**

```tsx
// ✅ Mobile-first SCSS
$breakpoints: (
  'sm': 640px,
  'md': 768px,
  'lg': 1024px,
  'xl': 1280px,
);

@mixin respond-to($breakpoint) {
  @media (min-width: map-get($breakpoints, $breakpoint)) {
    @content;
  }
}

// Container component
.container {
  padding: 16px; // Mobile padding

  @include respond-to('md') {
    padding: 24px;
    max-width: 768px;
  }

  @include respond-to('lg') {
    max-width: 1024px;
  }
}

// Grid layout
.grid {
  display: grid;
  gap: 16px;
  grid-template-columns: 1fr; // Mobile: single column

  @include respond-to('sm') {
    grid-template-columns: repeat(2, 1fr);
  }

  @include respond-to('md') {
    grid-template-columns: repeat(3, 1fr);
    gap: 24px;
  }
}

// ✅ Responsive image component
function ResponsiveImage({ src, alt }: { src: string; alt: string }) {
  return (
    <picture>
      {/* Mobile: small image */}
      <source media="(max-width: 640px)" srcSet={`${src}?w=320`} />
      {/* Tablet */}
      <source media="(max-width: 1024px)" srcSet={`${src}?w=768`} />
      {/* Desktop */}
      <img src={`${src}?w=1200`} alt={alt} loading="lazy" />
    </picture>
  );
}

// ✅ Mobile-safe touch interactions
const focusStyles = css`
  &:focus {
    outline: none;
    @media (prefers-reduced-motion: no-preference) {
      box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.5);
    }
  }

  @media (hover: none) {
    /* Touch device */
    &:active {
      background: rgba(0, 0, 0, 0.1);
    }
  }
`;

// ✅ Viewport units and safe-area-insets
.full-screen-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  padding-top: max(16px, env(safe-area-inset-top));
  padding-bottom: max(16px, env(safe-area-inset-bottom));

  @media (max-height: 500px) {
    padding: 8px;
  }
}

// ✅ Viewport meta tag
// In HTML head:
// <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=5.0" />

// ✅ React Hook for responsive design
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = React.useState(() => {
    // Check media query on client side only
    if (typeof window !== 'undefined') {
      return window.matchMedia(query).matches;
    }
    return false;
  });

  React.useEffect(() => {
    const mediaQueryList = window.matchMedia(query);

    const handleChange = (e: MediaQueryListEvent) => {
      setMatches(e.matches);
    };

    mediaQueryList.addEventListener('change', handleChange);
    return () => mediaQueryList.removeEventListener('change', handleChange);
  }, [query]);

  return matches;
}

// Usage
function ResponsiveComponent() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isDarkMode = useMediaQuery('(prefers-color-scheme: dark)');

  return (
    <div>
      {isMobile ? <MobileLayout /> : <DesktopLayout />}
      {isDarkMode && <style>{darkModeStyles}</style>}
    </div>
  );
}
```

**Interview tip:**
"Mobile-first approach ensures base styles work on all devices. Use semantic breakpoints, test on real devices, optimize images with srcset."

---

## ARCHITECTURE & SYSTEM DESIGN

### 14) How do you structure a large-scale React application with 50+ features?

**Short answer:**
Feature-based folder structure, shared UI library, separate API/Redux layers, lazy-load routes, enforce dependency rules.

**Code example:**

```
src/
  ├── app/
  │   ├── App.tsx
  │   ├── routes.tsx
  │   └── store.ts
  │
  ├── features/
  │   ├── auth/
  │   │   ├── components/
  │   │   │   ├── LoginForm.tsx
  │   │   │   └── ProtectedRoute.tsx
  │   │   ├── hooks/
  │   │   │   └── useAuth.ts
  │   │   ├── services/
  │   │   │   └── authService.ts
  │   │   ├── redux/
  │   │   │   ├── authSlice.ts
  │   │   │   └── authSelectors.ts
  │   │   ├── types/
  │   │   │   └── auth.types.ts
  │   │   └── index.ts (barrel export)
  │   │
  │   ├── posts/
  │   │   ├── components/
  │   │   ├── hooks/
  │   │   ├── services/
  │   │   ├── redux/
  │   │   ├── types/
  │   │   └── index.ts
  │   │
  │   └── [50 more features...]
  │
  ├── shared/
  │   ├── ui/
  │   │   ├── Button.tsx
  │   │   ├── Modal.tsx
  │   │   └── index.ts
  │   ├── hooks/
  │   │   ├── useAsync.ts
  │   │   └── useLocalStorage.ts
  │   ├── utils/
  │   │   ├── string.utils.ts
  │   │   └── date.utils.ts
  │   ├── types/
  │   │   └── common.types.ts
  │   └── api/
  │       ├── client.ts (Axios instance)
  │       └── interceptors.ts
  │
  └── styles/
      ├── variables.scss
      ├── global.scss
      └── mixins.scss
```

**Example files:**

```tsx
// features/auth/index.ts (barrel export for clean imports)
export * from './components/ProtectedRoute';
export * from './hooks/useAuth';
export { authSlice, selectUser, selectIsAuthenticated } from './redux/authSlice';

// features/auth/services/authService.ts
export class AuthService {
  static async login(email: string, password: string) {
    return axiosInstance.post('/auth/login', { email, password });
  }

  static async logout() {
    return axiosInstance.post('/auth/logout');
  }
}

// features/auth/hooks/useAuth.ts
export function useAuth() {
  const user = useAppSelector(selectUser);
  const isAuthenticated = useAppSelector(selectIsAuthenticated);
  const dispatch = useAppDispatch();

  const login = useCallback(async (email, password) => {
    const response = await AuthService.login(email, password);
    dispatch(authSlice.actions.setUser(response.data));
  }, [dispatch]);

  return { user, isAuthenticated, login };
}

// app/routes.tsx
export const routes = [
  {
    path: '/',
    element: <HomePage />,
  },
  {
    path: '/dashboard',
    element: <RequireAuth><Dashboard /></RequireAuth>,
  },
  {
    path: '/posts',
    element: <RequireAuth><PostsPage /></RequireAuth>,
  },
];

// app/store.ts
export const store = configureStore({
  reducer: {
    auth: authSlice.reducer,
    posts: postsSlice.reducer,
    // 50+ more slices
  },
});
```

**Interview tip:**
"Feature-based structure scales to 100+ features. Barrel exports hide implementation details. Use TypeScript paths for clean imports: `import { Button } from '@/shared/ui'`."

---

### 15) How do you implement a micro frontend architecture with React?

**Short answer:**
Use module federation (Webpack 5), separate builds per feature, compile micro apps independently, share common dependencies.

**Code example:**

```typescript
// apps/main-app/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'mainApp',
      filename: 'remoteEntry.js',
      remotes: {
        authApp: 'authApp@http://localhost:3001/remoteEntry.js',
        postsApp: 'postsApp@http://localhost:3002/remoteEntry.js',
      },
      shared: ['react', 'react-dom', '@reduxjs/toolkit'],
    }),
  ],
};

// apps/auth-app/webpack.config.js
new ModuleFederationPlugin({
  name: 'authApp',
  filename: 'remoteEntry.js',
  exposes: {
    './LoginPage': './src/pages/LoginPage.tsx',
    './useAuth': './src/hooks/useAuth.ts',
  },
  shared: ['react', 'react-dom', '@reduxjs/toolkit'],
})

// Main app usage
const LoginPage = React.lazy(() => import('authApp/LoginPage'));

export function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LoginPage />
    </Suspense>
  );
}
```

**Interview tip:**
"Micro frontends enable independent deployment. Module Federation (Webpack 5) handles dependency sharing elegantly. Consider complexity vs. benefit tradeoff."

---

## Interview Preparation Strategy

1. **React Advanced**: Master concurrent features, error boundaries, performance optimization
2. **Redux**: Know when to use Saga vs. Thunk, normalization patterns, middleware
3. **MSAL**: Understand OAuth 2.0 flow, token management, scope-based permissions
4. **Axios**: Implement robust interceptors, retry logic, error handling
5. **CSS**: Know scalable architectures, responsive design, accessibility

---

## Key Takeaways

- Performance: Profiling → lazy loading → memoization → virtualization
- State management: Normalize data, split contexts by frequency, consider external stores
- Authentication: Handle token refresh, MSAL + REDUX integration, secure storage
- API: Centralized Axios client, interceptors for auth, exponential backoff retries
- CSS: Component-based, mobile-first, themes with variables, accessibility compliance

---

## Practice Tasks

1. Build a multi-feature app with lazy loading and performance monitoring
2. Implement MSAL + Redux + Axios integration end-to-end
3. Create responsive design with Tailwind + CSS Modules comparison
4. Design error handling strategy with retries and fallbacks
5. Refactor monolithic app to feature-based architecture
