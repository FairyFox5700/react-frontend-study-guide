# React Frontend Study Guide

**Goal:** Learn how to design, build, test, and ship maintainable React applications—with special attention to frontend patterns and data flow.

**Recommended ratio:** 30% reading, 70% building.

---

## Contents

1. [The 30-second mental model](#1-the-30-second-mental-model)
2. [What to learn first](#2-what-to-learn-first)
3. [React data flow](#3-react-data-flow)
4. [Where each kind of state belongs](#4-where-each-kind-of-state-belongs)
5. [Core component patterns](#5-core-component-patterns)
6. [Effects, refs, and external systems](#6-effects-refs-and-external-systems)
7. [Data fetching and mutations](#7-data-fetching-and-mutations)
8. [Application architecture](#8-application-architecture)
9. [TypeScript patterns](#9-typescript-patterns)
10. [Routing, forms, errors, and accessibility](#10-routing-forms-errors-and-accessibility)
11. [Testing strategy](#11-testing-strategy)
12. [Performance](#12-performance)
13. [Eight-week learning roadmap](#13-eight-week-learning-roadmap)
14. [Progressive project cookbook](#14-progressive-project-cookbook)
15. [Debugging guide](#15-debugging-guide)
16. [Production checklist](#16-production-checklist)
17. [Resources](#17-resources)

---

# 1. The 30-second mental model

React is a library for expressing a user interface as a function of data:

```text
UI = render(props, state)
```

A React component:

1. receives **props** from its parent;
2. may remember **local state**;
3. returns a description of the UI;
4. reacts to user events by requesting state updates;
5. re-renders when its props, state, or consumed context changes.

The default React flow is:

```text
Parent state
    │
    ├── props ──▶ Child UI
    │                │
    │                └── user event
    │                       │
    ◀──── callback/action ──┘
    │
state update → re-render → updated UI
```

The most important rule is **one-way data flow**:

- data flows down;
- events flow up;
- state has one clear owner;
- derived values are calculated rather than stored again.

## React compared with C#/.NET

| React concept | Useful .NET analogy | Important difference |
|---|---|---|
| Component | Razor component or small view | A function component runs again to describe the next UI |
| Props | Constructor/method input | Props are read-only snapshots |
| State | Private mutable view-model data | Never mutate it directly; replace it through a setter/dispatch |
| Hook | Reusable stateful behavior | Hook call order must remain stable |
| Context | Scoped ambient dependency | Best for broadly shared values, not every piece of app state |
| Reducer | Pure command handler | Returns new state; it does not mutate the existing state |
| Effect | Synchronization with an external system | Not a general-purpose “after render” business-logic container |
| Render | Pure projection | Do not perform network calls or mutations during render |

---

# 2. What to learn first

## P0 — essential

- Modern JavaScript: modules, destructuring, spread, arrays, promises, `async`/`await`
- TypeScript fundamentals and discriminated unions
- JSX and function components
- Props, state, events, conditional rendering, lists, and keys
- Controlled inputs
- Lifting state up
- State ownership and derived state
- `useState`, `useReducer`, `useContext`, `useRef`, and `useEffect`
- Immutable updates
- Loading, empty, success, and error states
- Component tests based on user-visible behavior

## P1 — production skills

- Routing and URL state
- Server-state caching and invalidation
- Forms and validation
- Feature-based project structure
- Error boundaries
- Accessibility
- Authentication boundaries and API security
- Code splitting
- Performance measurement
- CI checks, production builds, and deployment

## P2 — learn after building real applications

- Suspense-enabled data loading
- Transitions and deferred rendering
- React Server Components
- Framework-specific server actions
- React Compiler and advanced memoization
- Design-system architecture
- Micro-frontends

Do not begin with P2. First learn to make state ownership and ordinary data flow obvious.

---

# 3. React data flow

## 3.1 Render flow

```text
State/props/context change
          │
          ▼
React calls component functions
          │
          ▼
Components calculate JSX
          │
          ▼
React compares the new and previous trees
          │
          ▼
React commits required DOM changes
          │
          ▼
Browser paints
          │
          ▼
Effects synchronize external systems
```

Rendering must stay pure. Given the same inputs, a component should calculate the same JSX and should not mutate external data.

## 3.2 Event flow

Events are where user-driven changes begin:

```tsx
function QuantityEditor() {
  const [quantity, setQuantity] = useState(1);

  function increment() {
    setQuantity(current => current + 1);
  }

  return (
    <button onClick={increment}>
      Quantity: {quantity}
    </button>
  );
}
```

Use the updater form, `setQuantity(current => current + 1)`, when the next value depends on the previous value.

## 3.3 Props down, events up

```tsx
type QuantityInputProps = {
  value: number;
  onChange: (next: number) => void;
};

function QuantityInput({ value, onChange }: QuantityInputProps) {
  return (
    <input
      aria-label="Quantity"
      type="number"
      min={1}
      value={value}
      onChange={event => onChange(Number(event.target.value))}
    />
  );
}

function OrderLine() {
  const [quantity, setQuantity] = useState(1);
  return <QuantityInput value={quantity} onChange={setQuantity} />;
}
```

`OrderLine` owns the state. `QuantityInput` is controlled by props. The child does not reach into the parent; it reports intent through `onChange`.

## 3.4 Derived state

Store the smallest possible state:

```tsx
const [items, setItems] = useState<CartItem[]>([]);

// Good: calculated during render.
const total = items.reduce(
  (sum, item) => sum + item.price * item.quantity,
  0,
);
```

Avoid this:

```tsx
const [items, setItems] = useState<CartItem[]>([]);
const [total, setTotal] = useState(0); // redundant

useEffect(() => {
  setTotal(calculateTotal(items));
}, [items]);
```

The second version creates two sources of truth and an unnecessary render.

## 3.5 State snapshots and batching

Each render sees a snapshot of state. Calling a setter requests another render; it does not rewrite the variable inside the current event handler.

```tsx
setCount(count + 1);
setCount(count + 1); // both read the same snapshot
```

For two increments:

```tsx
setCount(current => current + 1);
setCount(current => current + 1);
```

## 3.6 Keys control identity

Keys tell React which list item or component instance is which.

```tsx
{orders.map(order => (
  <OrderRow key={order.id} order={order} />
))}
```

Use stable domain IDs. Do not use array indexes when rows can move, be inserted, or be removed. Changing a component's key intentionally resets its local state.

---

# 4. Where each kind of state belongs

Use this decision table before adding a store.

| State kind | Examples | Best owner |
|---|---|---|
| Ephemeral UI state | Menu open, selected tab, draft text | Nearest component that needs it |
| Shared screen state | Filters used by sibling components | Closest common parent |
| Complex screen workflow | Checkout steps, editor transitions | `useReducer`, possibly plus context |
| URL state | Search, page, sort, selected record | Router/search parameters |
| Server state | Products, orders, user profile | Router loader or server-state library/cache |
| Broad app preference | Theme, locale, authenticated user summary | Context/provider |
| Non-rendering mutable value | Timer ID, DOM node, previous raw value | `useRef` |
| Form state | Input values, touched/errors, submission | Form component or form library |

## State placement algorithm

For each value:

1. Can it be calculated from props or existing state? **Calculate it.**
2. Does only one component need it? **Keep it local.**
3. Do siblings need it? **Lift it to their nearest common parent.**
4. Should refresh/back/forward preserve it? **Put it in the URL or persistence layer.**
5. Did it come from a server? **Treat it as cached server state.**
6. Is the transition logic complex? **Use a reducer.**
7. Is it genuinely global and stable? **Use context or an external store.**

## Avoid contradictory state

Bad:

```ts
const [isLoading, setIsLoading] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [hasError, setHasError] = useState(false);
```

Several flags can be true at once. Prefer one explicit state:

```ts
type Status =
  | { kind: 'idle' }
  | { kind: 'loading' }
  | { kind: 'success'; orderId: string }
  | { kind: 'error'; message: string };
```

This makes impossible states unrepresentable.

---

# 5. Core component patterns

## 5.1 Composition over configuration

Prefer normal composition:

```tsx
<Card>
  <CardHeader title="Order 123" />
  <OrderSummary order={order} />
  <CardActions>
    <Button>Refund</Button>
  </CardActions>
</Card>
```

This is usually clearer than one giant component with many Boolean props.

## 5.2 Controlled and uncontrolled components

A controlled component receives its important value and change handler:

```tsx
<SearchBox value={query} onValueChange={setQuery} />
```

An uncontrolled component owns its value and may expose only an initial value:

```tsx
<SearchBox defaultValue="shoes" />
```

Choose controlled when the parent must coordinate, validate, persist, or reset the value. Choose uncontrolled for isolated, reusable behavior where the parent does not need every change.

## 5.3 Reducer pattern

Use a reducer when many events update related state or when transitions deserve names.

```tsx
type CartState = {
  items: CartItem[];
  coupon?: string;
};

type CartAction =
  | { type: 'itemAdded'; item: CartItem }
  | { type: 'itemRemoved'; itemId: string }
  | { type: 'couponApplied'; code: string }
  | { type: 'cleared' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'itemAdded':
      return { ...state, items: [...state.items, action.item] };
    case 'itemRemoved':
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.itemId),
      };
    case 'couponApplied':
      return { ...state, coupon: action.code };
    case 'cleared':
      return { items: [] };
  }
}
```

Reducer rules:

- pure function;
- no API calls;
- no DOM access;
- no mutation of the old state;
- every action describes what happened;
- exhaustive action handling.

## 5.4 Context pattern

Use context to distribute a stable capability or value through a subtree: theme, locale, session summary, or reducer state/dispatch.

```tsx
const CartStateContext = createContext<CartState | null>(null);
const CartDispatchContext = createContext<Dispatch<CartAction> | null>(null);
```

Keep read and dispatch contexts separate when it helps reduce unnecessary consumers and clarifies dependencies.

Context is not automatically a replacement for all props or a complete server-state solution.

## 5.5 Custom hook pattern

A custom hook shares behavior, not UI:

```tsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const goOnline = () => setIsOnline(true);
    const goOffline = () => setIsOnline(false);

    window.addEventListener('online', goOnline);
    window.addEventListener('offline', goOffline);
    return () => {
      window.removeEventListener('online', goOnline);
      window.removeEventListener('offline', goOffline);
    };
  }, []);

  return isOnline;
}
```

A hook may use other hooks. Its name must start with `use`. Do not call hooks inside loops, conditions, or nested callbacks.

## 5.6 Container and presentational separation

Useful when a component becomes hard to test:

```text
OrdersPage (routing, data loading, commands)
└── OrdersView (rendering and user events)
    ├── OrderFilters
    ├── OrderTable
    └── Pagination
```

Do not split mechanically. Split when responsibilities, change frequency, or testing needs differ.

## 5.7 Adapter pattern at API boundaries

Do not let raw API shapes leak across the application:

```ts
type OrderDto = {
  order_id: string;
  total_cents: number;
};

type Order = {
  id: string;
  total: number;
};

function toOrder(dto: OrderDto): Order {
  return {
    id: dto.order_id,
    total: dto.total_cents / 100,
  };
}
```

Validate untrusted network data at the boundary. TypeScript types do not validate runtime JSON.

## 5.8 Compound component pattern

Good for coordinated UI such as tabs, menus, or accordions:

```tsx
<Tabs defaultValue="details">
  <Tabs.List>
    <Tabs.Trigger value="details">Details</Tabs.Trigger>
    <Tabs.Trigger value="history">History</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="details">...</Tabs.Panel>
  <Tabs.Panel value="history">...</Tabs.Panel>
</Tabs>
```

Use it when the parts form one conceptual widget. Ensure keyboard and ARIA behavior is correct.

## 5.9 State machine pattern

For workflows with strict transitions, model states and events explicitly:

```text
idle --SUBMIT--> submitting
submitting --RESOLVE--> success
submitting --REJECT--> error
error --RETRY--> submitting
```

A reducer may be enough. Use a state-machine library only when guards, parallel states, visualization, or complex orchestration justify it.

---

# 6. Effects, refs, and external systems

## 6.1 What an Effect is for

Use an Effect to synchronize React with something outside React:

- browser event subscription;
- WebSocket connection;
- media player;
- map widget;
- imperative third-party library;
- timer;
- network synchronization when the framework/data layer does not own it.

```tsx
useEffect(() => {
  const connection = createConnection(roomId);
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

Think: **start synchronization; later stop synchronization**.

## 6.2 What an Effect is not for

Usually avoid Effects for:

- deriving values used for rendering;
- handling a button click;
- resetting all child state when an identity changes—use a `key`;
- chaining business-state changes through several Effects;
- copying props into state without a clear reason.

## 6.3 Dependency rule

Every reactive value read by an Effect belongs in its dependency list. Do not silence the linter to force a desired lifecycle. Restructure the code instead.

## 6.4 Refs

A ref stores a value across renders without causing a render when changed.

Use it for:

- DOM element access;
- timer/observer handles;
- integration objects;
- mutable values that do not belong in the visual output.

If changing a value should update the screen, use state rather than a ref.

---

# 7. Data fetching and mutations

## 7.1 Separate four concepts

```text
UI state      → dialog open, current draft
URL state     → query, sort, page
Server state  → fetched orders and cache metadata
Mutation      → create/update/delete command
```

Mixing these together is a common source of complexity.

## 7.2 Request state model

Every request-driven screen needs explicit states:

```text
initial/loading
success with data
success but empty
recoverable error
refreshing/stale
```

Do not use one spinner to cover every condition.

## 7.3 Fetching safely in a basic client component

```tsx
useEffect(() => {
  const controller = new AbortController();

  async function load() {
    try {
      setState({ kind: 'loading' });
      const response = await fetch(`/api/orders?query=${encodeURIComponent(query)}`, {
        signal: controller.signal,
      });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const data: unknown = await response.json();
      setState({ kind: 'success', orders: parseOrders(data) });
    } catch (error) {
      if (!controller.signal.aborted) {
        setState({ kind: 'error', message: toErrorMessage(error) });
      }
    }
  }

  void load();
  return () => controller.abort();
}, [query]);
```

This is educational, but production applications often benefit from router loaders or a server-state library that handles caching, deduplication, retries, invalidation, and stale data.

## 7.4 Mutation flow

```text
User submits
   │
   ▼
Validate client input
   │
   ▼
Send command to API
   │
   ├── failure → retain input + show actionable error
   │
   └── success → update/invalidate cache → render server truth
```

The server must validate and authorize again. Client-side checks are usability, not security.

## 7.5 Optimistic updates

Use when success is likely and rollback is safe:

1. snapshot current cache;
2. show the expected result immediately;
3. send the request;
4. replace with server truth on success;
5. restore the snapshot and show an error on failure.

Avoid optimism for destructive, financial, or difficult-to-reverse actions unless the product behavior is carefully designed.

## 7.6 Race conditions

Typing a search quickly may produce requests A, B, and C. If A finishes last, it must not overwrite C. Use cancellation, request identity, or a data layer that handles stale results.

---

# 8. Application architecture

## 8.1 Recommended feature-based structure

```text
src/
├── app/
│   ├── App.tsx
│   ├── routes.tsx
│   └── providers.tsx
├── features/
│   ├── orders/
│   │   ├── api/
│   │   │   ├── ordersApi.ts
│   │   │   └── orderSchemas.ts
│   │   ├── components/
│   │   │   ├── OrderFilters.tsx
│   │   │   └── OrderTable.tsx
│   │   ├── hooks/
│   │   ├── model/
│   │   │   ├── order.ts
│   │   │   └── orderReducer.ts
│   │   ├── pages/
│   │   │   └── OrdersPage.tsx
│   │   └── tests/
│   └── checkout/
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── lib/
│   └── types/
└── main.tsx
```

Organize code primarily by business capability, then by technical role inside the feature. Avoid one global `components/` directory containing the entire application.

## 8.2 Dependency direction

```text
app shell
   │
   ├── feature modules
   │      │
   │      ├── feature UI
   │      ├── feature model
   │      └── API adapter
   │
   └── shared primitives
```

Rules:

- shared code must not import a feature;
- one feature should not reach into another feature's internals;
- API details stop at adapters;
- UI components do not import environment secrets;
- dependencies should point toward stable abstractions.

## 8.3 End-to-end data path

```text
Browser URL
   │
Router parses search params
   │
Loader/query fetches `/api/orders`
   │
API adapter validates + maps DTOs
   │
Cache stores server state
   │
OrdersPage selects required data
   │
Props flow into OrderTable
   │
User clicks “Cancel”
   │
Event handler invokes mutation
   │
Server authorizes and updates
   │
Relevant cache entry invalidated
   │
Fresh server truth renders
```

## 8.4 Dependency injection without a framework

Pass capabilities through props or narrow context:

```tsx
type OrderService = {
  cancel(id: string): Promise<void>;
};

function OrdersPage({ orderService }: { orderService: OrderService }) {
  // Easy to replace in tests.
}
```

Do not build a large service-container abstraction unless the application actually needs it.

---

# 9. TypeScript patterns

## Component props

```tsx
type ButtonProps = {
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
};
```

Prefer narrow domain types over broad strings.

## Discriminated unions

```ts
type Loadable<T> =
  | { kind: 'idle' }
  | { kind: 'loading' }
  | { kind: 'success'; data: T }
  | { kind: 'error'; error: string };
```

Then render exhaustively:

```tsx
switch (orders.kind) {
  case 'idle': return null;
  case 'loading': return <Spinner />;
  case 'success': return <OrderTable orders={orders.data} />;
  case 'error': return <ErrorMessage>{orders.error}</ErrorMessage>;
}
```

## Boundary validation

Treat API JSON, local storage, URL parameters, and `postMessage` payloads as `unknown` until validated.

## Avoid these shortcuts

- `any` for convenience;
- non-null assertions everywhere;
- giant prop interfaces reused across unrelated components;
- raw DTO types used as domain models;
- Boolean props that permit contradictory combinations.

---

# 10. Routing, forms, errors, and accessibility

## Routing

Use the URL for user-visible navigation state:

```text
/orders?status=pending&sort=created-desc&page=2
```

Benefits:

- shareable links;
- refresh preservation;
- correct browser back/forward behavior;
- easier analytics and debugging.

## Forms

A form needs:

- labels connected to controls;
- typed form values;
- client and server validation;
- field-level and form-level errors;
- pending state;
- prevention of accidental duplicate submission;
- focus management after errors;
- retention of entered data on recoverable failure.

Use native form semantics before adding custom keyboard behavior.

## Error boundaries

Error boundaries catch rendering failures below them. Place them around meaningful recovery regions—not only once around the whole application.

They do not replace handling expected API errors in normal request state.

## Accessibility baseline

- semantic HTML first;
- every input has a label;
- everything interactive works with a keyboard;
- focus is visible;
- dialogs trap and restore focus correctly;
- use ARIA only when native HTML cannot express the behavior;
- announce important async status changes appropriately;
- maintain usable color contrast;
- test with keyboard and a screen reader.

---

# 11. Testing strategy

## Test pyramid

```text
          Few end-to-end tests
       More integration tests
   Many focused unit/component tests
```

## What to test

### Pure unit tests

- reducers;
- formatters;
- validators;
- adapters;
- calculation functions.

### Component/integration tests

- user sees correct loading/empty/error/success UI;
- typing and clicking produces visible behavior;
- forms validate and submit;
- providers and routes work together;
- API behavior is simulated at the network boundary.

### End-to-end tests

- critical revenue or business flows;
- authentication and permissions;
- route navigation;
- full create/update/cancel workflows.

## Test behavior, not implementation

Prefer:

```tsx
await user.click(screen.getByRole('button', { name: /place order/i }));
expect(await screen.findByText(/order confirmed/i)).toBeVisible();
```

Avoid asserting hook internals, private state, or component implementation details.

## Deliberate failure exercise

Make the API return `500` while submitting an order. Verify:

- the button does not remain permanently disabled;
- the user's form values remain;
- an actionable error appears;
- retry succeeds;
- no duplicate order is created.

---

# 12. Performance

Start with measurement, not memoization.

## Priority order

1. Avoid unnecessary network waterfalls.
2. Keep state near the components that need it.
3. Do not store redundant derived data.
4. Virtualize genuinely large lists.
5. Split large route bundles.
6. Use the React Profiler and browser performance tools.
7. Add memoization only where measurement shows value.

`useMemo` and `useCallback` are performance tools, not correctness tools. Incorrect code should not require them to work.

For non-urgent updates, transitions or deferred values can keep urgent input responsive. Use them after understanding ordinary state flow.

---

# 13. Eight-week learning roadmap

Assume 7–10 hours per week. Each week ends with working software.

## Week 1 — JavaScript, TypeScript, JSX

**Learn**

- modules, arrays, object updates, closures;
- promises and `async`/`await`;
- TypeScript unions, narrowing, generics;
- JSX, props, lists, keys, conditional rendering.

**Build:** Static product catalog from an in-memory array.

**Deliberate failure:** Use duplicate keys and observe incorrect identity/warnings; then fix with stable IDs.

**Done when**

- [ ] No `any`
- [ ] Components have typed props
- [ ] Lists use stable keys
- [ ] `npm run build` succeeds

**Read:** React Quick Start and TypeScript JSX documentation.

## Week 2 — State and one-way data flow

**Learn**

- `useState`;
- state snapshots;
- controlled inputs;
- lifting state;
- derived state;
- immutable updates.

**Build:** Searchable, sortable product table with a cart counter.

**Deliberate failure:** Store both filtered products and the filter inputs in state; observe synchronization problems; remove the redundant state.

**Done when**

- [ ] Search and sort are controlled
- [ ] Total is derived, not stored
- [ ] State has a clear owner
- [ ] Refresh behavior is understood

**Read:** Thinking in React, Choosing the State Structure, Sharing State.

## Week 3 — Reducers, context, and custom hooks

**Learn**

- reducer actions and pure transitions;
- context boundaries;
- custom hooks;
- component composition.

**Build:** Multi-step cart and checkout draft.

**Deliberate failure:** Introduce contradictory `isLoading`/`isSuccess` flags; replace them with a discriminated union.

**Done when**

- [ ] Reducer tests cover every action
- [ ] Context is limited to the checkout subtree
- [ ] UI components do not mutate state objects
- [ ] A custom hook contains reusable behavior rather than markup

## Week 4 — Effects and API data

**Learn**

- synchronization model for Effects;
- cleanup;
- cancellation and race conditions;
- request states;
- DTO validation and mapping.

**Build:** Orders dashboard backed by a mock API.

**Deliberate failure:** Return search responses out of order; prevent stale responses from replacing current results.

**Done when**

- [ ] Loading, empty, error, and success states exist
- [ ] Requests can be cancelled or superseded
- [ ] API data is validated at the boundary
- [ ] No Effect exists only to derive render data

## Week 5 — Routing and forms

**Learn**

- route layout;
- route parameters and search parameters;
- form validation;
- pending and error UX;
- focus management.

**Build:** Order list, details, and create-order routes.

**Deliberate failure:** Simulate a validation error and a server error. Preserve valid input and focus the first invalid field.

**Done when**

- [ ] Filters survive refresh through the URL
- [ ] Browser back/forward works
- [ ] Duplicate submission is prevented
- [ ] Server errors are distinguishable from field errors

## Week 6 — Testing and accessibility

**Learn**

- reducer/unit tests;
- React Testing Library;
- network-level API mocking;
- keyboard and semantic HTML testing.

**Build:** Automated test suite for the orders application.

**Deliberate failure:** Refactor internal component structure; behavior tests should continue passing.

**Done when**

- [ ] Critical behavior is tested via roles and accessible names
- [ ] No test reads private component state
- [ ] Keyboard-only flow works
- [ ] Error and retry paths are tested

## Week 7 — Architecture and server-state patterns

**Learn**

- feature folders;
- server state versus client state;
- cache invalidation;
- optimistic updates;
- dependency boundaries.

**Build:** Add cancel/refund mutations and cached order details.

**Deliberate failure:** Reject an optimistic update. Roll it back and display the correct status.

**Done when**

- [ ] Features do not import each other's internals
- [ ] Mutations invalidate or update the correct cache
- [ ] Server truth eventually wins
- [ ] API concerns stay behind an adapter

## Week 8 — Production readiness

**Learn**

- bundle inspection;
- route-level code splitting;
- error boundaries;
- environment configuration;
- observability and deployment.

**Build:** Deploy the complete application.

**Deliberate failure:** Break an API endpoint and a lazy-loaded route. Verify graceful recovery and diagnostic evidence.

**Done when**

- [ ] Typecheck, lint, test, and production build pass
- [ ] No secret is present in the browser bundle
- [ ] Errors are observable without exposing personal data
- [ ] README reproduces setup from a clean clone
- [ ] Critical workflow is covered by an end-to-end test

---

# 14. Progressive project cookbook

## Project A — Product explorer

**Time:** 4–6 hours  
**Level:** beginner

**Build:** Search, category filter, stock filter, sort, and product details.

**Ingredients:** Vite, React, TypeScript, static JSON.

**Recipe**

1. Sketch the component tree.
2. Identify minimal state.
3. Build the static version first.
4. Add controlled filters.
5. Derive the visible products during render.
6. Add tests for combinations of filters.

**Deliberate failure:** Duplicate filtered data in state.

**Completion proof:** A test changes two filters and confirms the visible products without testing implementation details.

## Project B — Order workflow

**Time:** 10–14 hours  
**Level:** intermediate

**Build:** List, details, create, cancel, and retry flows using a mock API.

**Ingredients:** Router, API mock, React Testing Library.

**Recipe**

1. Put page/filter state into the URL.
2. Define request-state unions.
3. Validate API responses.
4. Implement list and details routes.
5. Add a form with pending and error states.
6. Add mutation invalidation.
7. Test errors and race conditions.

**Deliberate failure:** Make an old search response arrive after a new one.

**Completion proof:** The URL is shareable, stale results do not replace current results, and errors can be retried.

## Project C — Production commerce dashboard

**Time:** 20–30 hours  
**Level:** advanced

**Build:** Authenticated dashboard with role-based actions, real API integration, charts, audit history, and deployment.

**Ingredients:** React framework or router, TypeScript, schema validation, server-state solution, test runner, end-to-end tests.

**Recipe**

1. Define feature boundaries and API contracts.
2. Build an end-to-end read path.
3. Add authenticated mutations.
4. Add cache invalidation and one safe optimistic flow.
5. Add nested recovery boundaries.
6. Measure bundle and rendering performance.
7. Add observability with privacy-safe metadata.
8. Deploy through CI.

**Deliberate failure:** Return `401`, `403`, `409`, `429`, and `500`; design distinct recovery behavior.

**Completion proof:** A clean clone can install, test, build, and run from README instructions; critical flows pass in CI.

---

# 15. Debugging guide

| Symptom | Likely cause | First check |
|---|---|---|
| UI shows an old value after `setState` | State is a render snapshot | Log in the next render; use functional updater if based on previous state |
| Infinite render loop | State update during render or Effect cycle | Find setter calls executed unconditionally |
| Effect runs repeatedly | Unstable object/function dependency | Move construction, narrow dependencies, or redesign synchronization |
| Input loses text | Component key changes | Inspect keys and conditional tree identity |
| Two components disagree | Duplicated state | Find the single owner and derive the rest |
| Wrong row updates | Unstable/index key | Use domain ID |
| Old search replaces new results | Request race | Cancel or identify stale requests |
| Context update re-renders too much | Provider value changes broadly | Split contexts, reduce provider scope, measure |
| Test breaks after harmless refactor | Testing implementation details | Query and assert user-visible behavior |
| Vite build passes despite type errors | Vite transpiles TS but does not typecheck | Run `tsc --noEmit` separately |

## Five-step debugging loop

1. Reproduce the smallest failing case.
2. Identify the owner of the incorrect state.
3. Trace the event that changes it.
4. Inspect render inputs, not only DOM output.
5. Add a regression test before refactoring.

---

# 16. Production checklist

## Correctness

- [ ] State has a clear owner
- [ ] Derived data is not duplicated
- [ ] Reducers are pure
- [ ] Loading, empty, error, success, and retry states exist
- [ ] Race conditions and cancellation are handled
- [ ] Every list uses stable keys

## Type and code quality

- [ ] TypeScript strict mode enabled
- [ ] No unexplained `any` or non-null assertions
- [ ] Runtime validation at external boundaries
- [ ] Lint and hook rules pass
- [ ] Feature dependencies follow the intended direction

## Tests

- [ ] Pure logic has unit tests
- [ ] Important components have behavior tests
- [ ] Critical workflows have end-to-end coverage
- [ ] Failure and retry paths are tested
- [ ] Tests do not depend on private component details

## Accessibility

- [ ] Keyboard-only workflow succeeds
- [ ] Forms have labels and useful errors
- [ ] Focus moves intentionally after navigation/errors/dialogs
- [ ] Interactive controls have accessible names
- [ ] Contrast and zoom are usable

## Security and privacy

- [ ] Authorization is enforced on the server
- [ ] No secret is shipped to the browser
- [ ] User-generated content is not injected as unsafe HTML
- [ ] Logs and analytics exclude sensitive data
- [ ] Dependencies and lockfile are reviewed

## Performance and operations

- [ ] Production build is inspected
- [ ] Large routes are split where useful
- [ ] Network waterfalls are minimized
- [ ] Errors are observable
- [ ] Source maps follow the organization's exposure policy
- [ ] Rollback and incident ownership are known

## Reproducibility

- [ ] Clean clone can install, typecheck, lint, test, and build
- [ ] Required environment variables are documented
- [ ] Example environment file contains placeholders only
- [ ] Supported runtime/package-manager versions are documented

---

# 17. Resources

Use official documentation as the source of truth. Complete one section, then reproduce it in your own project.

## React core

1. [React Quick Start](https://react.dev/learn)
2. [Thinking in React](https://react.dev/learn/thinking-in-react)
3. [Managing State](https://react.dev/learn/managing-state)
4. [Choosing the State Structure](https://react.dev/learn/choosing-the-state-structure)
5. [Sharing State Between Components](https://react.dev/learn/sharing-state-between-components)
6. [Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context)
7. [Escape Hatches](https://react.dev/learn/escape-hatches)
8. [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
9. [Lifecycle of Reactive Effects](https://react.dev/learn/lifecycle-of-reactive-effects)
10. [Suspense reference](https://react.dev/reference/react/Suspense)

## Project setup and TypeScript

11. [Creating a React App](https://react.dev/learn/creating-a-react-app)
12. [Vite Guide](https://vite.dev/guide/)
13. [Vite TypeScript behavior](https://vite.dev/guide/features.html#typescript)
14. [Vite production build](https://vite.dev/guide/build)
15. [TypeScript JSX documentation](https://www.typescriptlang.org/docs/handbook/jsx.html)

For learning the React fundamentals, a small Vite + TypeScript SPA keeps the moving parts visible. For a production application, evaluate a React framework/router that owns routing, data loading, code splitting, and rendering strategy.

## Testing

16. [React Testing Library introduction](https://testing-library.com/docs/react-testing-library/intro/)
17. [React Testing Library example](https://testing-library.com/docs/react-testing-library/example-intro/)

## Recommended study loop

For every topic:

1. read one official page;
2. reproduce its idea without copy/paste;
3. deliberately break it;
4. explain the resulting behavior in your own words;
5. write one test;
6. apply the pattern to the current project.

---

## Final principle

When React code feels complicated, ask these questions in order:

1. What is the smallest source of truth?
2. Who owns it?
3. Is this value state, derived data, URL state, or server state?
4. Which event changes it?
5. Can the component remain pure?
6. Is an Effect really synchronizing an external system?
7. Can a user-visible test prove the behavior?

Clear answers to those questions solve most frontend data-flow problems before another library is needed.
