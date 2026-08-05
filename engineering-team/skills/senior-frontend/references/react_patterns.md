# React Patterns

Practical, production-grade React patterns for building maintainable component libraries and application code. This reference goes deeper than the quick examples in `SKILL.md` — use it when you're deciding *which* pattern fits a given problem, not just how to write one.

## 1. Compound Components

**When to use:** a parent owns shared state/behavior and several children need to read or mutate it without prop-drilling — tabs, accordions, menus, selects, form groups.

**When NOT to use:** if there's only one child that needs the state, just pass props. Compound components add indirection that isn't worth it for a single consumer.

```tsx
import { createContext, useContext, useId, useState } from 'react';

interface TabsContextValue {
  activeId: string;
  setActiveId: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.* components must be rendered inside <Tabs>');
  return ctx;
}

interface TabsProps {
  defaultActiveId: string;
  children: React.ReactNode;
}

export function Tabs({ defaultActiveId, children }: TabsProps) {
  const [activeId, setActiveId] = useState(defaultActiveId);
  return (
    <TabsContext.Provider value={{ activeId, setActiveId }}>
      <div role="tablist">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeId, setActiveId } = useTabsContext();
  const isActive = activeId === id;
  return (
    <button
      role="tab"
      aria-selected={isActive}
      tabIndex={isActive ? 0 : -1}
      onClick={() => setActiveId(id)}
    >
      {children}
    </button>
  );
}

function Panel({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeId } = useTabsContext();
  if (activeId !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

Tabs.Tab = Tab;
Tabs.Panel = Panel;

// Usage — the consumer controls composition and ordering freely
<Tabs defaultActiveId="profile">
  <Tabs.Tab id="profile">Profile</Tabs.Tab>
  <Tabs.Tab id="billing">Billing</Tabs.Tab>
  <Tabs.Panel id="profile">...</Tabs.Panel>
  <Tabs.Panel id="billing">...</Tabs.Panel>
</Tabs>
```

**Failure mode to avoid:** throwing the context provider value object inline (`value={{ activeId, setActiveId }}`) creates a new object every render, which invalidates `React.memo` on every consumer even when `activeId` didn't change. Memoize it:

```tsx
const value = useMemo(() => ({ activeId, setActiveId }), [activeId]);
```

`setActiveId` from `useState` is stable across renders, so this memo only recomputes when `activeId` actually changes.

## 2. Render Props vs. Hooks

Render props (function-as-child, or a `render` prop) were the pre-hooks way to share stateful *and rendering* logic. Hooks replaced most of their use cases, but render props still win when the consumer needs to control **markup structure**, not just data.

```tsx
// Render prop: the caller controls the DOM structure entirely
function Toggle({ children }: { children: (api: { on: boolean; toggle: () => void }) => React.ReactNode }) {
  const [on, setOn] = useState(false);
  return <>{children({ on, toggle: () => setOn(o => !o) })}</>;
}

<Toggle>
  {({ on, toggle }) => (
    <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>
  )}
</Toggle>
```

```tsx
// Hook equivalent: the caller controls the DOM too, with less nesting and no "wrapper hell"
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return { on, toggle };
}

function MyComponent() {
  const { on, toggle } = useToggle();
  return <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>;
}
```

**Decision rule:**
- Need to share *stateful logic* that a component consumes directly → **custom hook**. It composes better (you can call multiple hooks in one component) and avoids the "wrapper hell" of nested render-prop components (`<A>{a => <B>{b => <C>{c => ...}}}</B>}</A>`).
- Need to share logic *plus* control injection points inside a component you don't own (e.g., a headless UI library exposing internal list virtualization state to arbitrary row markup) → **render props**, because hooks can't be used by an author who doesn't control the calling component's JSX. This is why libraries like `downshift` and `react-table` (v7) expose render-prop / prop-getter APIs even in a hooks-first era — the consumer's markup is unknowable in advance.
- Libraries increasingly ship **both**: a hook for logic (`useCombobox`) and let consumers build their own JSX, which is strictly more flexible than a render prop because it composes with other hooks.

## 3. State Colocation

The single highest-leverage rule for render performance: **state should live as close as possible to where it's read**, and no closer to the root than necessary.

Anti-pattern — lifting state to the top for no reason:

```tsx
// BAD: every keystroke in the search box re-renders the entire dashboard,
// including charts, tables, and sidebars that don't care about `query`.
function Dashboard() {
  const [query, setQuery] = useState('');
  return (
    <div>
      <SearchBox value={query} onChange={setQuery} />
      <ExpensiveChart />
      <ExpensiveTable />
    </div>
  );
}
```

Fix — colocate the state inside the component that owns it, or push the subtree that depends on it down:

```tsx
// GOOD: state and its only consumer live together; re-renders are scoped.
function Dashboard() {
  return (
    <div>
      <SearchBox />
      <ExpensiveChart />
      <ExpensiveTable />
    </div>
  );
}

function SearchBox() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

If sibling components genuinely need the derived value (e.g., filtered results), colocate the state one level up at the **lowest common ancestor**, not at the app root. This is the "lift state up, but only as far as necessary" rule from the official React docs, and it's the difference between a component tree that re-renders 3 nodes on interaction versus 300.

**Composition as a performance pattern:** you can keep state high (for cross-cutting reasons) but still avoid re-rendering unrelated subtrees by passing components as `children`/props instead of rendering them internally:

```tsx
// The re-render caused by `count` changing does NOT re-render <ExpensiveTree/>,
// because it was created in the parent's render scope and passed down as a prop —
// React bails out on re-rendering an element it already has a reference to.
function Counter({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}
    </div>
  );
}

function App() {
  return (
    <Counter>
      <ExpensiveTree />
    </Counter>
  );
}
```

## 4. Context Performance Pitfalls

Context is a *dependency injection* mechanism, not a state management library. Its biggest footgun: **every consumer of a context re-renders whenever the provider's value changes, regardless of which field the consumer actually reads.**

```tsx
// BAD: ThemeContext and UserContext are unrelated concerns bundled into one
// value object. A theme toggle re-renders every component that reads `user`.
const AppContext = createContext({ theme: 'light', user: null });
```

Fixes, in order of preference:

1. **Split contexts by concern and by change-frequency.** `ThemeContext` and `UserContext` should be separate providers so a theme change doesn't re-render user-dependent trees.
2. **Split "state" context from "dispatch" context.** Setter functions from `useState`/`useReducer` are referentially stable, so a `DispatchContext` consumer never re-renders due to state changes:

```tsx
const CountStateContext = createContext(0);
const CountDispatchContext = createContext<React.Dispatch<Action> | null>(null);

function CountProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, 0);
  return (
    <CountStateContext.Provider value={state}>
      <CountDispatchContext.Provider value={dispatch}>
        {children}
      </CountDispatchContext.Provider>
    </CountStateContext.Provider>
  );
}

// A component that only dispatches actions (e.g. an increment button)
// never re-renders when `state` changes, because it only subscribes
// to CountDispatchContext, whose value never changes identity.
function IncrementButton() {
  const dispatch = useContext(CountDispatchContext)!;
  return <button onClick={() => dispatch({ type: 'increment' })}>+</button>;
}
```

3. **Memoize the provider value.** `value={{ ...}}` inline creates a new object every render of the provider component, forcing every consumer to re-render even if the logical values didn't change:

```tsx
const value = useMemo(() => ({ state, dispatch }), [state]); // dispatch is stable, omit or include, doesn't matter
```

4. **For high-frequency updates (mouse position, scroll offset, form field values across 50+ fields), don't use Context at all.** Use an external store (Zustand, Jotai) with selector-based subscriptions, or a ref + imperative updates, so only the specific component reading that slice re-renders. Context has no selector mechanism — `useContext` is all-or-nothing.
5. **Consider `useSyncExternalStore`** when you need context-like global access but with the granular subscription semantics an external store gives you — this is what Zustand/Jotai build on under the hood, and it's the React-sanctioned way to subscribe to a value outside React's render cycle without tearing.

## 5. Suspense and Concurrent Patterns

### Suspense for data (not just lazy components)

```tsx
import { Suspense } from 'react';

function ProductPage({ id }: { id: string }) {
  return (
    <div>
      <ProductHeader id={id} />
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews id={id} />
      </Suspense>
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations id={id} />
      </Suspense>
    </div>
  );
}
```

Two independent `Suspense` boundaries let `Reviews` and `Recommendations` stream in on their own schedules instead of one slow fetch blocking the whole page (see the streaming SSR section of `nextjs_optimization_guide.md` for the server-rendering half of this story). Each boundary should wrap the *smallest* unit of UI that can meaningfully show a loading state — wrapping the entire page in one `Suspense` throws away the benefit.

### `useTransition` for non-blocking updates

Use when a state update triggers expensive rendering work (e.g., filtering 10,000 rows) and you don't want it to block the input from feeling responsive:

```tsx
function SearchableList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [filtered, setFiltered] = useState(items);

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value); // urgent: keep the input snappy
    startTransition(() => {
      // non-urgent: can be interrupted by the next keystroke
      setFiltered(items.filter(i => i.name.includes(e.target.value)));
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        {filtered.map(i => <Row key={i.id} item={i} />)}
      </div>
    </>
  );
}
```

Without `useTransition`, every keystroke synchronously re-renders the filtered list before the next keystroke can be processed, producing visible input lag once the list is a few thousand items. With it, React keeps the input responsive and lets the filtered-list render be interrupted and restarted with fresher input.

### `useDeferredValue` for expensive derived renders

Same underlying mechanism as `useTransition`, but for values you receive rather than state you set (e.g., a prop, or a value from a store):

```tsx
function SearchResults({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => expensiveSearch(deferredQuery), [deferredQuery]);
  return <ResultsList results={results} />;
}
```

### `use()` (React 19) for reading promises/context conditionally

```tsx
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  // Unlike hooks, `use` can be called after early returns / inside conditionals
  const comments = use(commentsPromise);
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}

function Page({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  return (
    <Suspense fallback={<Spinner />}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

The promise must be created above the component (e.g., in a Server Component or a cache-wrapped fetcher) and passed down — creating it inline inside the client component would refetch on every render.

## 6. `React.memo`, `useMemo`, `useCallback` — when they actually help

These only pay off when **all** of these are true:
1. The component is expensive to render (measured, not assumed — use React DevTools Profiler).
2. It re-renders often with the same props.
3. Props can be made referentially stable (primitives, or memoized objects/functions).

```tsx
const ExpensiveRow = memo(function ExpensiveRow({ item, onSelect }: RowProps) {
  return <tr onClick={() => onSelect(item.id)}>{/* heavy render */}</tr>;
});

function List({ items }: { items: Item[] }) {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  // Stable identity across renders — required for memo() on ExpensiveRow to pay off
  const handleSelect = useCallback((id: string) => setSelectedId(id), []);

  return (
    <table>
      {items.map(item => (
        <ExpensiveRow key={item.id} item={item} onSelect={handleSelect} />
      ))}
    </table>
  );
}
```

Common mistake: wrapping a component in `memo()` while still passing an inline arrow function or object literal as a prop. The memo does nothing because the prop reference changes every render anyway — you have to fix the whole prop chain, not just the leaf.

## 7. Error Boundaries + Suspense together

```tsx
<ErrorBoundary FallbackComponent={ErrorFallback}>
  <Suspense fallback={<Skeleton />}>
    <DataView />
  </Suspense>
</ErrorBoundary>
```

Suspense handles the loading state; error boundaries handle the rejected-promise/thrown-error state. They're orthogonal — one Suspense boundary can sit inside or outside an error boundary depending on whether you want a retry to reset just the failed subtree or the whole page.
