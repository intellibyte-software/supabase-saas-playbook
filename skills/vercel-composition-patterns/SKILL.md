---
name: vercel-composition-patterns
description: React composition patterns that scale. Use when refactoring components with boolean prop proliferation, building flexible component libraries, or designing reusable APIs. Triggers on tasks involving compound components, render props, context providers, or component architecture. Includes React 19 API changes.
---

# React Composition Patterns

Composition patterns for building flexible, maintainable React components. Avoid boolean prop proliferation by using compound components, lifting state, and composing internals.

## When to Apply

Reference these guidelines when:

- Refactoring components with many boolean props
- Building reusable component libraries
- Designing flexible component APIs
- Reviewing component architecture
- Working with compound components or context providers

## Rule Categories by Priority

| Priority | Category | Impact | Key Patterns |
|----------|----------|--------|--------------|
| 1 | Component Architecture | HIGH | Avoid boolean props, compound components |
| 2 | State Management | MEDIUM | Decouple implementation, context interface, lift state |
| 3 | Implementation Patterns | MEDIUM | Explicit variants, children over render props |
| 4 | React 19 APIs | MEDIUM | No forwardRef, use() hook |

## 1. Component Architecture (HIGH)

### Avoid Boolean Props

**Problem:** Boolean props create combinatorial complexity that's hard to maintain.

```typescript
// BAD: Boolean prop proliferation
<Card isCompact isHighlighted hasBorder isLoading />
```

**Solution:** Use composition and explicit variants:

```typescript
// GOOD: Compose behavior
<Card variant="compact" className="highlighted">
   <Card.Header>Title</Card.Header>
   <Card.Body>Content</Card.Body>
</Card>
```

### Compound Components

Structure complex components with shared context:

```typescript
const TabsContext = createContext<TabsState | null>(null);

function Tabs({ children, defaultValue }: TabsProps) {
   const [active, setActive] = useState(defaultValue);
   return (
      <TabsContext.Provider value={{ active, setActive }}>
         {children}
      </TabsContext.Provider>
   );
}

Tabs.List = function TabsList({ children }) { /* ... */ };
Tabs.Tab = function Tab({ value, children }) { /* ... */ };
Tabs.Panel = function TabsPanel({ value, children }) { /* ... */ };

// Usage
<Tabs defaultValue="tab1">
   <Tabs.List>
      <Tabs.Tab value="tab1">First</Tabs.Tab>
      <Tabs.Tab value="tab2">Second</Tabs.Tab>
   </Tabs.List>
   <Tabs.Panel value="tab1">Content 1</Tabs.Panel>
   <Tabs.Panel value="tab2">Content 2</Tabs.Panel>
</Tabs>
```

## 2. State Management (MEDIUM)

### Decouple Implementation

The provider is the only place that knows how state is managed:

```typescript
// The context consumer doesn't know if state comes from
// useState, useReducer, Zustand, or an API
interface CounterState {
   count: number;
   increment: () => void;
   decrement: () => void;
}

function CounterProvider({ children }: { children: ReactNode }) {
   const [count, setCount] = useState(0);
   return (
      <CounterContext.Provider value={{
         count,
         increment: () => setCount(c => c + 1),
         decrement: () => setCount(c => c - 1),
      }}>
         {children}
      </CounterContext.Provider>
   );
}
```

### Context Interface Pattern

Define a generic interface with state, actions, and meta:

```typescript
interface ContextValue<T> {
   state: T;
   actions: Record<string, (...args: any[]) => void>;
   meta: { isLoading: boolean; error: Error | null };
}
```

### Lift State

Move state into provider components for sibling access:

```typescript
// BAD: Prop drilling through intermediaries
<Parent count={count} onIncrement={increment}>
   <Child count={count} onIncrement={increment} />
</Parent>

// GOOD: State in provider, consumed where needed
<CounterProvider>
   <Display />     {/* reads count */}
   <Controls />    {/* calls increment */}
</CounterProvider>
```

## 3. Implementation Patterns (MEDIUM)

### Explicit Variants

Create explicit variant components instead of boolean modes:

```typescript
// BAD
<Button isPrimary isLarge isOutline />

// GOOD
<Button variant="primary" size="lg" style="outline" />

// BETTER for very different behaviors
<PrimaryButton size="lg" />
<OutlineButton size="lg" />
```

### Children Over Render Props

Use children for composition instead of `renderX` props:

```typescript
// BAD: render prop
<List renderItem={(item) => <ItemRow item={item} />} />

// GOOD: children composition
<List>
   {items.map(item => <ItemRow key={item.id} item={item} />)}
</List>
```

## 4. React 19 APIs (MEDIUM)

> React 19+ only. Skip if using React 18 or earlier.

### No forwardRef

React 19 passes `ref` as a regular prop. No need for `forwardRef`:

```typescript
// React 18 (old)
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => (
   <input ref={ref} {...props} />
));

// React 19 (new)
function Input({ ref, ...props }: InputProps & { ref?: Ref<HTMLInputElement> }) {
   return <input ref={ref} {...props} />;
}
```

### use() Hook

Replace `useContext()` with `use()`:

```typescript
// React 18
const value = useContext(MyContext);

// React 19
const value = use(MyContext);
```

`use()` also works with promises for Suspense-compatible data fetching.
