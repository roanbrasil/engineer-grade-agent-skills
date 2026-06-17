---
name: react-patterns
description: Expert React 18+ patterns — invoked when designing components, hooks, state management, data fetching, or performance optimization in React applications.
---

# React Patterns (React 18+)

## Component Design

### Single Responsibility

```tsx
// BAD: one component doing too much
function UserPage({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [posts, setPosts] = useState<Post[]>([]);
  // ...all the fetching, all the rendering, all the side effects
}

// GOOD: split by concern
function UserPage({ userId }: { userId: string }) {
  return (
    <div>
      <UserProfile userId={userId} />
      <UserPostFeed userId={userId} />
    </div>
  );
}
```

### Props Interface

```tsx
// Every component gets an explicit interface
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  // 'children' comes from ComponentPropsWithoutRef<'button'>
}

function Button({
  variant = 'primary',
  size = 'md',
  loading = false,
  disabled,
  children,
  ...rest
}: ButtonProps) {
  return (
    <button
      disabled={disabled || loading}
      aria-disabled={disabled || loading}
      {...rest}
    >
      {loading ? <Spinner aria-hidden /> : null}
      {children}
    </button>
  );
}
```

### Component Composition

```tsx
// children prop: most flexible composition
function Panel({ title, children }: { title: string; children: ReactNode }) {
  return (
    <section aria-labelledby="panel-title">
      <h2 id="panel-title">{title}</h2>
      <div>{children}</div>
    </section>
  );
}

// Render prop: inject logic from parent
function Toggle({ render }: { render: (on: boolean, toggle: () => void) => ReactNode }) {
  const [on, setOn] = useState(false);
  return <>{render(on, () => setOn(o => !o))}</>;
}

// Usage:
<Toggle render={(on, toggle) => (
  <button onClick={toggle}>{on ? 'Hide' : 'Show'} details</button>
)} />
```

---

## Hooks Patterns

### useState vs useReducer

```tsx
// useState: simple, independent values
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// useReducer: complex state with multiple transitions
type CartAction =
  | { type: 'ADD_ITEM'; item: CartItem }
  | { type: 'REMOVE_ITEM'; id: string }
  | { type: 'UPDATE_QUANTITY'; id: string; qty: number }
  | { type: 'CLEAR' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.item] };
    case 'REMOVE_ITEM':
      return { ...state, items: state.items.filter(i => i.id !== action.id) };
    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: state.items.map(i =>
          i.id === action.id ? { ...i, quantity: action.qty } : i
        ),
      };
    case 'CLEAR':
      return { items: [] };
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });
  // dispatch({ type: 'ADD_ITEM', item })
}
```

### useEffect Rules

```tsx
// Rule 1: dependency array must be complete
function UserAvatar({ userId }: { userId: string }) {
  const [avatar, setAvatar] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    fetchAvatar(userId).then(url => {
      if (!cancelled) setAvatar(url);
    });
    // Rule 2: always cleanup subscriptions and async ops
    return () => { cancelled = true; };
  }, [userId]); // Rule 3: userId changes → effect re-runs

  return avatar ? <img src={avatar} alt="" /> : <Skeleton />;
}

// Rule 4: useLayoutEffect for DOM measurements (synchronous)
function Tooltip({ target, children }: TooltipProps) {
  const tooltipRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const rect = target.getBoundingClientRect();
    // Position tooltip before browser paints — no flicker
    tooltipRef.current!.style.top = `${rect.bottom + 8}px`;
    tooltipRef.current!.style.left = `${rect.left}px`;
  }, [target]);

  return <div ref={tooltipRef}>{children}</div>;
}
```

### Custom Hooks

```tsx
// useDebounce: delays value update
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// useLocalStorage: persisted state
function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    setStored(prev => {
      const next = typeof value === 'function' ? (value as (p: T) => T)(prev) : value;
      window.localStorage.setItem(key, JSON.stringify(next));
      return next;
    });
  }, [key]);

  return [stored, setValue] as const;
}

// useIntersectionObserver: lazy loading / infinite scroll trigger
function useIntersectionObserver(options?: IntersectionObserverInit) {
  const ref = useRef<HTMLElement>(null);
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => setIsIntersecting(entry.isIntersecting),
      options
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, [options]);

  return [ref, isIntersecting] as const;
}
```

### useMemo / useCallback — When to Use

```tsx
// useMemo: expensive computation only
// NOT for object identity; profiler must show it's needed
function DataTable({ rows, filters }: DataTableProps) {
  // Only memoize if filtering 10k+ rows is measurably slow
  const filteredRows = useMemo(
    () => rows.filter(row => matchesFilters(row, filters)),
    [rows, filters]
  );

  return <Table rows={filteredRows} />;
}

// useCallback: stable function reference for React.memo children
function SearchPage() {
  const [query, setQuery] = useState('');
  const [page, setPage] = useState(1);

  // Stable ref — SearchResults is wrapped in React.memo
  const handlePageChange = useCallback((newPage: number) => {
    setPage(newPage);
    window.scrollTo(0, 0);
  }, []); // no deps — setPage is always stable

  return <SearchResults query={query} page={page} onPageChange={handlePageChange} />;
}

// React.memo: skip re-render if props are shallowly equal
const SearchResults = memo(function SearchResults({
  query,
  page,
  onPageChange
}: SearchResultsProps) {
  // Only re-renders when query, page, or onPageChange reference changes
  return <div>...</div>;
});
```

---

## Performance

```
Render decision tree:

Parent re-renders
       ↓
Is child wrapped in React.memo?
  NO  → always re-renders
  YES → shallow compare props
           ↓
    Props changed?
      NO  → skip render ✓
      YES → re-renders
              ↓
         Are objects/functions new references each time?
           YES → memo is useless; add useMemo/useCallback
```

### Code Splitting

```tsx
// Route-level splitting
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings  = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// Feature-level splitting (heavy component loaded on demand)
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));

function PostEditor({ isEditing }: { isEditing: boolean }) {
  return isEditing ? (
    <Suspense fallback={<EditorSkeleton />}>
      <RichTextEditor />
    </Suspense>
  ) : null;
}
```

### Virtual Lists

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // estimated row height in px
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflowY: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              width: '100%',
            }}
          >
            <ListItem item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## State Management

```
Decision matrix:

Where is the state used?
  └─ One component           → useState
  └─ Parent + 1-2 children   → useState + props
  └─ Distant relatives       →
       Changes frequently?
         YES → Zustand
         NO  → Context API
  └─ Data from server/API    → TanStack Query
  └─ Complex, many consumers → Redux Toolkit
  └─ URL-derived             → URL search params (nuqs)
```

### Context API — Low-Frequency State Only

```tsx
// Good: theme, auth user, locale — change rarely
interface AuthContextValue {
  user: User | null;
  isLoading: boolean;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const { data: user, isLoading } = useQuery({ queryKey: ['me'], queryFn: fetchMe });
  const { mutate: logout } = useMutation({ mutationFn: logoutApi });

  return (
    <AuthContext.Provider value={{ user: user ?? null, isLoading, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// BAD: never put high-frequency state (selected row, hover state) in context
// Every consumer re-renders on every change
```

### Zustand

```tsx
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  total: () => number;
}

const useCartStore = create<CartStore>()(
  devtools((set, get) => ({
    items: [],

    addItem: (item) =>
      set(state => ({ items: [...state.items, item] }), false, 'addItem'),

    removeItem: (id) =>
      set(state => ({ items: state.items.filter(i => i.id !== id) }), false, 'removeItem'),

    // Derived value: computed, not stored
    total: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
  }))
);

// Component subscribes to slice — only re-renders if that slice changes
function CartCount() {
  const count = useCartStore(state => state.items.length);
  return <span>{count}</span>;
}
```

---

## Data Fetching

### TanStack Query

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// useQuery: read data
function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 min before background refetch
    retry: 2,
  });

  if (isLoading) return <ProfileSkeleton />;
  if (isError) return <ErrorMessage error={error} />;

  return <UserCard user={data} />;
}

// useMutation: write data with optimistic update
function FollowButton({ targetUserId }: { targetUserId: string }) {
  const queryClient = useQueryClient();

  const follow = useMutation({
    mutationFn: (id: string) => followUser(id),
    onMutate: async (id) => {
      // Cancel in-flight queries for this user
      await queryClient.cancelQueries({ queryKey: ['user', id] });
      // Snapshot previous value
      const previous = queryClient.getQueryData(['user', id]);
      // Optimistically update
      queryClient.setQueryData(['user', id], (old: User) => ({
        ...old,
        isFollowing: true,
        followerCount: old.followerCount + 1,
      }));
      return { previous };
    },
    onError: (_err, id, context) => {
      // Rollback on error
      queryClient.setQueryData(['user', id], context?.previous);
    },
    onSettled: (_, __, id) => {
      // Always refetch to sync with server
      queryClient.invalidateQueries({ queryKey: ['user', id] });
    },
  });

  return <button onClick={() => follow.mutate(targetUserId)}>Follow</button>;
}

// useInfiniteQuery: paginated / infinite scroll
function PostFeed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => fetchPosts({ cursor: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });

  const posts = data?.pages.flatMap(p => p.posts) ?? [];

  return (
    <>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load more'}
        </button>
      )}
    </>
  );
}
```

### Why Not useEffect + useState for Fetching

```tsx
// BAD: race conditions, no deduplication, no caching
function BadUserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetchUser(userId).then(u => {
      setUser(u);  // RACE: if userId changes fast, old response wins
      setLoading(false);
    });
  }, [userId]);
  // No error handling, no cancellation, no caching, no deduplication
}
```

---

## Compound Component Pattern (Full Example)

```tsx
// Tabs: compound components with context
const TabsContext = createContext<{
  activeTab: string;
  setActiveTab: (id: string) => void;
} | null>(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Must be used within <Tabs>');
  return ctx;
}

function Tabs({ defaultTab, children }: { defaultTab: string; children: ReactNode }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.List = function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Tab = function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab, setActiveTab } = useTabs();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab } = useTabs();
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage:
<Tabs defaultTab="overview">
  <Tabs.List>
    <Tabs.Tab id="overview">Overview</Tabs.Tab>
    <Tabs.Tab id="analytics">Analytics</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="overview"><OverviewContent /></Tabs.Panel>
  <Tabs.Panel id="analytics"><AnalyticsContent /></Tabs.Panel>
</Tabs>
```

---

## Error Boundaries

```tsx
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<
  { fallback: ReactNode; children: ReactNode },
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    reportError(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// Wrap routes and critical sections
function App() {
  return (
    <ErrorBoundary fallback={<AppCrashScreen />}>
      <Router>
        <ErrorBoundary fallback={<WidgetError />}>
          <PricingWidget />
        </ErrorBoundary>
      </Router>
    </ErrorBoundary>
  );
}
```

---

## Anti-Patterns

- **Deriving state from props in useState**: `useState(props.value)` — only runs once; use `useMemo` or compute directly
- **Missing cleanup in useEffect**: subscriptions leak; always return cleanup function
- **Putting everything in Context**: causes entire tree to re-render on any state change
- **`useCallback` on every function**: adds overhead without React.memo children; measure first
- **useEffect for data fetching**: race conditions, no caching — use TanStack Query
- **Prop drilling > 2 levels**: pass through context or lift to state manager
- **Key=index in lists**: breaks reconciliation when items reorder or insert; use stable IDs

---

## Quick Reference Checklist

- [ ] Each component has a single visual responsibility
- [ ] TypeScript interface for all props; extend native element props with `ComponentPropsWithoutRef`
- [ ] `useEffect` has complete dependency array and cleanup function
- [ ] Custom hooks for any reusable stateful logic (name starts with `use`)
- [ ] `useMemo` / `useCallback` added only after profiler confirms unnecessary renders
- [ ] `React.memo` wrapping components that receive stable prop references
- [ ] `React.lazy` + `Suspense` for route-level and heavy feature splits
- [ ] Virtual list for 500+ item lists
- [ ] TanStack Query for all server state; no `useEffect` data fetching
- [ ] Zustand for client state shared across distant components
- [ ] Context only for infrequently-changing values (theme, auth, locale)
- [ ] Error boundaries around routes and critical widgets
- [ ] Compound components for complex multi-part UI (Tabs, Select, Accordion)
