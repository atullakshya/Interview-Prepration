# Redux Intermediate Interview Q&A

This guide is for developers with 2-5 years of React + Redux experience.
Each question includes explanations, code with comments, and interview tips.

---

## 1) What is Redux, and why would you use it?

**Short answer:**
Redux is a predictable state management library that centralizes app state, making it easier to debug and maintain complex state changes across components.

**Why interviewers ask:**
They want to confirm you understand when Redux is necessary and its core value proposition.

**Explanation:**
Redux solves **prop drilling** and makes state changes **traceable and predictable**. Instead of passing data through many components, Redux keeps a single source of truth.

**Code example:**

```tsx
import { configureStore, createSlice } from '@reduxjs/toolkit';
import { useDispatch, useSelector } from 'react-redux';

// Action + Reducer combined in one "slice"
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    // Action: increment
    increment: (state) => {
      state.value += 1; // Immer handles immutability
    },
    // Action: decrement
    decrement: (state) => {
      state.value -= 1;
    },
  },
});

// Export actions for dispatch
export const { increment, decrement } = counterSlice.actions;

// Create store
const store = configureStore({
  reducer: {
    counter: counterSlice.reducer,
  },
});

// Component using Redux
function Counter() {
  const dispatch = useDispatch();
  const count = useSelector((state) => state.counter.value); // Select state

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
    </div>
  );
}
```

**Interview tip:**
Say: "Redux is best when you have complex cross-component state or high-frequency updates that need a clear audit trail."

---

## 2) What are Actions, Reducers, and Store?

**Short answer:**
- **Action**: Plain object describing what happened
- **Reducer**: Pure function that updates state based on action
- **Store**: Centralized container holding entire state tree

**Code example:**

```tsx
import { configureStore } from '@reduxjs/toolkit';

// 1. ACTION: Describes an event
const ADD_TODO = 'ADD_TODO'; // action type
const addTodo = (text) => ({
  type: ADD_TODO,
  payload: text, // data associated with action
});

// 2. REDUCER: Pure function that transforms state
const initialState = {
  todos: [],
};

function todoReducer(state = initialState, action) {
  switch (action.type) {
    case ADD_TODO:
      // Must return new state, not mutate
      return {
        ...state,
        todos: [...state.todos, { id: Date.now(), text: action.payload }],
      };
    default:
      return state;
  }
}

// 3. STORE: Holds the entire state tree
const store = configureStore({
  reducer: todoReducer,
});

// Dispatch an action
store.dispatch(addTodo('Learn Redux'));

// Get current state
console.log(store.getState());
// Output: { todos: [{ id: 1234567890, text: 'Learn Redux' }] }
```

**Interview tip:**
Emphasize: "Actions are pure data objects, reducers are pure functions, and the store is the single source of truth."

---

## 3) What is Redux Thunk, and how does it handle async actions?

**Short answer:**
Redux Thunk is middleware that lets you dispatch functions (thunks) instead of plain actions, allowing async operations like API calls.

**Why it matters:**
Without thunk, you cannot dispatch async operations. Thunk enables delayed or conditional dispatches.

**Code example:**

```tsx
import { configureStore, createSlice } from '@reduxjs/toolkit';
import thunk from 'redux-thunk';

const postsSlice = createSlice({
  name: 'posts',
  initialState: {
    data: [],
    loading: false,
    error: null,
  },
  reducers: {
    // Synchronous actions
    setLoading: (state, action) => {
      state.loading = action.payload;
    },
    setPostsSuccess: (state, action) => {
      state.data = action.payload;
      state.loading = false;
      state.error = null;
    },
    setPostsError: (state, action) => {
      state.error = action.payload;
      state.loading = false;
    },
  },
});

export const { setLoading, setPostsSuccess, setPostsError } = postsSlice.actions;

// THUNK: Async action creator (function that returns a function)
export const fetchPosts = () => async (dispatch) => {
  dispatch(setLoading(true));
  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts');
    const data = await response.json();
    dispatch(setPostsSuccess(data));
  } catch (error) {
    dispatch(setPostsError(error.message));
  }
};

const store = configureStore({
  reducer: {
    posts: postsSlice.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(thunk), // Enable thunk
});

// Usage in component
function PostsContainer() {
  const dispatch = useDispatch();
  const { data, loading, error } = useSelector((state) => state.posts);

  useEffect(() => {
    // Dispatch thunk (function gets invoked by thunk middleware)
    dispatch(fetchPosts());
  }, [dispatch]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <ul>{data.map((post) => <li key={post.id}>{post.title}</li>)}</ul>;
}
```

**Interview tip:**
"Thunk middleware intercepts function actions, calls them with dispatch and getState, enabling async workflows."

---

## 4) What is the difference between Connect and Hooks (useSelector, useDispatch)?

**Short answer:**
- **connect()**: Class-based HOC, older pattern, requires mapStateToProps/mapDispatchToProps
- **useSelector/useDispatch**: Modern hooks, simpler, preferred in most new code

**Code example (Connect pattern):**

```tsx
import { connect } from 'react-redux';

// Old way: withConnect HOC

function Counter(props) {
  return (
    <div>
      <p>Count: {props.count}</p>
      <button onClick={() => props.increment()}>Increment</button>
    </div>
  );
}

// Map state to props
const mapStateToProps = (state) => ({
  count: state.counter.value,
});

// Map dispatch to props
const mapDispatchToProps = (dispatch) => ({
  increment: () => dispatch(increment()),
});

export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```

**Code example (Hooks pattern - preferred):**

```tsx
import { useSelector, useDispatch } from 'react-redux';

function Counter() {
  // Directly select state
  const count = useSelector((state) => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch(increment())}>Increment</button>
    </div>
  );
}
```

**Interview tip:**
"Hooks are simpler, avoid wrapper HOCs, and integrate better with other hooks. Use them in new projects."

---

## 5) What are Selectors, and why use them?

**Short answer:**
Selectors are pure functions that extract and compute state values. They prevent direct state coupling and enable memoization.

**Why:**
- Encapsulate state shape
- Recompute only when dependencies change
- Easier refactoring

**Code example:**

```tsx
import { useSelector } from 'react-redux';

// Basic selector
export const selectAllTodos = (state) => state.todos.items;

// Computed selector (derived state)
export const selectCompleteTodos = (state) =>
  state.todos.items.filter((todo) => todo.completed);

// Selector with parameter
export const selectTodoById = (state, id) =>
  state.todos.items.find((todo) => todo.id === id);

// Using reselect for memoization (optional but recommended)
import { createSelector } from 'reselect';

// Without memoization, this reruns every time component renders
// even if todos haven't changed
const selectCompleteTodosNoMemo = (state) =>
  state.todos.items.filter((todo) => todo.completed);

// With memoization: only recomputes if state.todos.items actually changes
export const selectCompleteTodosMemo = createSelector(
  (state) => state.todos.items, // input selector
  (items) => items.filter((todo) => todo.completed) // output selector
);

// Component
function TodoList() {
  // Using selector
  const completedTodos = useSelector(selectCompleteTodosMemo);

  return (
    <ul>
      {completedTodos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

**Interview tip:**
"Selectors keep your state shape private and enable performance optimization through memoization."

---

## 6) How do you handle normalized state in Redux?

**Short answer:**
Normalize state by storing entities in separate, flat objects indexed by ID instead of nested arrays. This prevents data duplication and simplifies updates.

**Bad (denormalized):**
```tsx
{
  posts: [
    {
      id: 1,
      title: 'First',
      author: { id: 101, name: 'Alice' },
      comments: [
        { id: 1001, text: 'Great!', authorId: 102 },
      ],
    },
  ],
}
```

**Good (normalized):**
```tsx
{
  entities: {
    posts: {
      1: { id: 1, title: 'First', authorId: 101 },
    },
    authors: {
      101: { id: 101, name: 'Alice' },
    },
    comments: {
      1001: { id: 1001, text: 'Great!', authorId: 102, postId: 1 },
    },
  },
  result: [1], // IDs of posts
}
```

**Code example with normalizr library:**

```tsx
import { normalize, schema } from 'normalizr';

// Define schema
const authorSchema = new schema.Entity('authors');
const commentSchema = new schema.Entity('comments');
const postSchema = new schema.Entity('posts', {
  author: authorSchema,
  comments: [commentSchema],
});

// API response
const apiResponse = {
  posts: [
    {
      id: 1,
      title: 'First',
      author: { id: 101, name: 'Alice' },
      comments: [{ id: 1001, text: 'Great!', authorId: 102 }],
    },
  ],
};

// Normalize it
const normalizedData = normalize(apiResponse.posts, [postSchema]);
console.log(normalizedData);
// Output:
// {
//   entities: {
//     authors: { 101: { id: 101, name: 'Alice' } },
//     posts: { 1: { id: 1, title: 'First', authorId: 101, ... } },
//     comments: { 1001: { id: 1001, text: 'Great!', ... } }
//   },
//   result: [1]
// }

// Reducer
const postsSlice = createSlice({
  name: 'posts',
  initialState: { byId: {}, allIds: [] },
  reducers: {
    // Normalized state makes updates simple
    updatePost: (state, action) => {
      const { id, changes } = action.payload;
      state.byId[id] = { ...state.byId[id], ...changes }; // Direct update
    },
  },
});
```

**Interview tip:**
"Normalization prevents duplication and makes single-point updates. Essential for large datasets with relationships."

---

## 7) What is middleware, and how do you create custom middleware?

**Short answer:**
Middleware intercepts actions before they reach reducers, enabling logging, async handling, or side effects.

**Code example:**

```tsx
import { configureStore } from '@reduxjs/toolkit';

// Custom middleware: logs all actions
const loggerMiddleware = (store) => (next) => (action) => {
  console.log('Dispatching:', action);
  const result = next(action); // Call the next middleware or reducer
  console.log('State after:', store.getState());
  return result;
};

// Custom middleware: catches errors
const errorMiddleware = (store) => (next) => (action) => {
  try {
    return next(action);
  } catch (error) {
    console.error('Error in action:', action, error);
    throw error;
  }
};

// Apply middleware to store
const store = configureStore({
  reducer: counterSlice.reducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware()
      .concat(loggerMiddleware)
      .concat(errorMiddleware),
});

// Usage
store.dispatch({ type: 'INCREMENT' });
// Console output:
// Dispatching: { type: 'INCREMENT' }
// State after: { value: 1 }
```

**Interview tip:**
"Middleware is where side effects live. Use for logging, error handling, crash analytics, etc."

---

## 8) What is Redux DevTools, and how do you use it?

**Short answer:**
Redux DevTools is a browser extension that lets you inspect every action, see state changes, and time-travel debug.

**Code setup:**

```tsx
import { configureStore } from '@reduxjs/toolkit';

const store = configureStore({
  reducer: {
    counter: counterSlice.reducer,
    posts: postsSlice.reducer,
  },
  // DevTools is automatically enabled in development
  // No extra setup needed with configureStore!
});

// Optional: if using legacy Redux, manual setup:
// const store = createStore(
//   rootReducer,
//   window.__REDUX_DEVTOOLS_EXTENSION__ &&
//     window.__REDUX_DEVTOOLS_EXTENSION__()
// );
```

**How to use:**
1. Install Redux DevTools browser extension
2. Open DevTools in browser (F12)
3. Click "Redux" tab
4. See every action dispatched, state changes
5. Click actions to time-travel to previous states
6. Use Inspector to check state shape

**Interview tip:**
"DevTools turns Redux into a time-traveling debugger—invaluable for tracking down bugs in complex state flows."

---

## 9) What is Redux Toolkit (RTK), and how does it simplify Redux?

**Short answer:**
Redux Toolkit provides utilities (createSlice, configureStore) that reduce boilerplate and add best practices like Immer for immutability.

**Before RTK (verbose):**

```tsx
const INCREMENT = 'counter/INCREMENT';

const initialState = { value: 0 };

function counterReducer(state = initialState, action) {
  switch (action.type) {
    case INCREMENT:
      return { ...state, value: state.value + 1 };
    default:
      return state;
  }
}

const increment = () => ({ type: INCREMENT });
```

**With RTK (concise):**

```tsx
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    // Automatically creates action + reducer
    increment: (state) => {
      state.value += 1; // Looks mutative, but Immer handles it
    },
  },
});

export const { increment } = counterSlice.actions;
export default counterSlice.reducer;
```

**Key RTK features:**
- `createSlice`: Combines action type, action creator, reducer
- `configureStore`: Sets up store with good defaults
- Immer integration: Write "mutative" code, actually immutable
- Built-in DevTools

**Interview tip:**
"RTK is the modern standard. It removes Redux boilerplate and prevents common mistakes."

---

## 10) What are the most common Redux mistakes?

**Short answer:**
1. Mutating state directly
2. Not normalizing nested data
3. Creating too much granular state
4. Overusing Redux for local component state

**Code examples:**

```tsx
// ❌ MISTAKE 1: Direct mutation
const reducer = (state, action) => {
  state.items.push(action.payload); // WRONG!
  return state;
};

// ✅ CORRECT: Return new state
const reducer = (state, action) => {
  return {
    ...state,
    items: [...state.items, action.payload],
  };
};

// ❌ MISTAKE 2: Bloated state
const initialState = {
  user: {
    id: 1,
    name: 'Alice',
    posts: [{ id: 1, title: '...' }], // Nested - hard to update
    comments: [{ id: 1, text: '...' }],
  },
};

// ✅ CORRECT: Flat, normalized
const initialState = {
  users: { 1: { id: 1, name: 'Alice' } },
  posts: { 1: { id: 1, title: '...' } },
  comments: { 1: { id: 1, text: '...' } },
};

// ❌ MISTAKE 3: Too many slices
const store = configureStore({
  reducer: {
    formField1: formField1Reducer,
    formField2: formField2Reducer,
    // ... 50 more tiny reducers
  },
});

// ✅ CORRECT: Group related state
const store = configureStore({
  reducer: {
    form: formReducer, // All form state together
    ui: uiReducer,
    data: dataReducer,
  },
});

// ❌ MISTAKE 4: Redux for everything
// Don't use Redux for:
// - Form input values (local state is fine)
// - UI state like open/close modals
// - Temporary UI filters

// ✅ CORRECT: Use Redux only for:
// - Shared global state (auth, theme, user data)
// - State needed by many components
// - State with complex logic
```

**Interview tip:**
"Redux is powerful but not a silver bullet. Use it for shared state, not local component UI."

---

## 11) How do you test Redux slices and selectors?

**Short answer:**
Test reducers and selectors as pure functions: pass input state and action, verify output.

**Code example:**

```tsx
import { configureStore, createSlice } from '@reduxjs/toolkit';

// Slice to test
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
  },
});

// Test reducer (pure function test)
describe('counter reducer', () => {
  it('should increment', () => {
    const initialState = { value: 0 };
    const action = counterSlice.actions.increment();
    const newState = counterSlice.reducer(initialState, action);
    expect(newState.value).toBe(1);
  });

  it('should decrement', () => {
    const state = { value: 5 };
    const action = counterSlice.actions.decrement();
    const newState = counterSlice.reducer(state, action);
    expect(newState.value).toBe(4);
  });
});

// Selector to test
export const selectCounterValue = (state) => state.counter.value;

describe('counter selector', () => {
  it('should select counter value', () => {
    const state = { counter: { value: 42 } };
    expect(selectCounterValue(state)).toBe(42);
  });
});

// Integration test with store
describe('counter integration', () => {
  it('should dispatch and update store', () => {
    const store = configureStore({
      reducer: { counter: counterSlice.reducer },
    });

    store.dispatch(counterSlice.actions.increment());
    expect(selectCounterValue(store.getState())).toBe(1);

    store.dispatch(counterSlice.actions.decrement());
    expect(selectCounterValue(store.getState())).toBe(0);
  });
});
```

**Interview tip:**
"Redux reducers are pure functions—test them like you'd test any pure function (input → output)."

---

## 12) How do you structure a large Redux application?

**Recommended structure:**

```txt
src/
  redux/
    slices/
      authSlice.ts    # Auth state
      postsSlice.ts   # Posts state
      uiSlice.ts      # UI toggles
    selectors/
      authSelectors.ts
      postsSelectors.ts
    middleware/
      apiMiddleware.ts
      loggerMiddleware.ts
    store.ts          # Create and export store
  features/
    Auth/
      AuthPage.tsx
      useAuth.ts      # Custom hook using selectors
    Posts/
      PostList.tsx
      PostCard.tsx
```

**Example (store setup):**

```tsx
// redux/store.ts
import { configureStore } from '@reduxjs/toolkit';
import authSlice from './slices/authSlice';
import postsSlice from './slices/postsSlice';
import uiSlice from './slices/uiSlice';

export const store = configureStore({
  reducer: {
    auth: authSlice,
    posts: postsSlice,
    ui: uiSlice,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Custom hooks for type safety
import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

**Interview tip:**
"Organize by feature and separation of concerns. Use custom typed hooks for DX."

---

## Rapid Fire Questions

1. What's the difference between action.type and action.payload?
2. How would you persist Redux state to localStorage?
3. What is the purpose of the getState function in thunks?
4. When should you use Redux vs. Context API?
5. How do you handle form state in Redux?

---

## Key Takeaways

- Redux centralizes state for complex apps
- Actions describe events, reducers transform state
- Thunks enable async actions
- Normalization prevents data duplication
- Redux Toolkit reduces boilerplate
- DevTools enables time-travel debugging
- Use Redux for shared state, not every piece of state

---

## How to Practice With This File

1. Pick 4 questions and answer them aloud in 2 minutes each.
2. Implement two small Redux features (counter + todo list).
3. Compare Redux vs React Context for a given scenario.
4. Practice Redux DevTools: dispatch actions and inspect state.
5. Build a small async app with thunks and loading states.
