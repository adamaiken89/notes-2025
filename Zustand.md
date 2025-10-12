# React & Zustand

## Common Problems in state management

### [Zombie Child Problem](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children)


> "Zombie child" refers specifically to the case where:
>
> Multiple nested connected components are mounted in a first pass, causing a child component to subscribe to the store before its parent
> An action is dispatched that deletes data from the store, such as a todo item
> The parent component would stop rendering that child as a result
> However, because the child subscribed first, its subscription runs before the parent stops rendering it. When it reads a value from the store based on props, that data no longer exists, and if the extraction logic is not careful, this may result in an error being thrown.
>
> *from Redux Official Document*

Although the parent component has destroyed, the child component is still left in the memory

Consequences:

- Memory leak
- Inconsistent UI states

### [React Concurrency](https://github.com/bvaughn/rfcs/blob/useMutableSource/text/0000-use-mutable-source.md)

React Reconciler

<https://www.youtube.com/watch?v=CGpMlWVcHok> Building a Custom React Renderer | Sophie Alpert

Prefer `Render-as-You-Fetch` with Suspense

<https://ithelp.ithome.com.tw/articles/10360013> 三種 React 資料請求策略：從 fetch-on-render 到 render-as-you-fetch

### [Context Loss](https://github.com/facebook/react/issues/13332)

## Why Zustand

- Be avoid to prop drilling
- Simplify the redux flow setup

## How to use Zustand

## Quick Example

State Model & Render Optimization

```typescript
import { create } from 'zustand'

type State = {
  count: number
}

type Actions = {
  increment: (qty: number) => void
  decrement: (qty: number) => void
}

const useCountStore = create<State & Actions>()((set) => ({
  count: 0,
  increment: (qty: number) => set((state) => ({ count: state.count + qty })),
  decrement: (qty: number) => set((state) => ({ count: state.count - qty })),
}))

function Counter() {
  const count = useCountStore((state) => state.count)
  return <h1>{count}</h1>
}

function Controls() {
  const increasePopulation = useCountStore((state) => state.increment)
  return <button onClick={increasePopulation}>one up</button>
}
```

- With types, recommend to have `State` and `Actions` to isolate variables and methods

on redux way, can simply use a single dispatch function

```typescript
$header = <header>I am a header</header>
```

start with `$` for DOM Element

## Challenges

<https://zustand.docs.pmnd.rs/guides/flux-inspired-practice> Zustand Documentation

<https://www.youtube.com/watch?v=1Fi4hK7L1ec> Zustand with Context API – An Advanced Pattern
<https://www.youtube.com/watch?v=6tEQ1nJZ51w> 5 Zustand BEST Practices in 5 Minutes
<https://www.youtube.com/watch?v=_ngCLZ5Iz-0> Zustand - Complete Tutorial

<https://tkdodo.eu/blog/working-with-zustand> #1 Working with Zustand
<https://tkdodo.eu/blog/zustand-and-react-context> #2 Zustand and React Context
<https://www.youtube.com/watch?v=QTZTUrAbjeo> Combining Zustand with React Query
<https://www.frontendundefined.com/posts/monthly/zustand-review> Why is Zustand a community favorite?

Reference hooks line as functions
-> Ensure not importing whole hooks to reduce performance issue

State Selector <- just get what you need here
Atomic Stable Selector
Object selector will create a new object and generate infinite update -> useShadow comparison

State & Action Separation
Store should be SMALL

### Merge State

```typescript
set((state) => ({ ...state, count: state.count + 1 }))

set((state) => ({ count: state.count + 1 }))
```

Update object state with ImmerJS

```typescript
immerInc: () =>
  set(produce((state: State) => { ++state.deep.nested.obj.count })),
```

Disable merging

```typescript
set((state) => newState, true)
```

### With no actions

```typescript
export const useBoundStore = create(() => ({
  count: 0,
  text: 'hello',
}))

export const inc = () =>
  useBoundStore.setState((state) => ({ count: state.count + 1 }))

export const setText = (text) => useBoundStore.setState({ text })
```

This has a few advantages:

It doesn't require a hook to call an action;
It facilitates code splitting.

### Slice Pattern

Slices do not have any create wrapped

```typescript
export const createFishSlice = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
})

export const createBearSlice = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
})

// update other stores
export const createBearFishSlice = (set, get) => ({
  addBearAndFish: () => {
    get().addBear()
    get().addFish()
  },
})
```

```typescript
import { create } from 'zustand'
import { createBearSlice } from './bearSlice'
import { createFishSlice } from './fishSlice'

export const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
  ...createBearFishSlice(...a), // spread the first one is enough
}))

```

Combined Store

- Do not `persist` inside slices

```typescript
import { create } from 'zustand'
import { createBearSlice } from './bearSlice'
import { createFishSlice } from './fishSlice'
import { persist } from 'zustand/middleware'

export const useBoundStore = create(
  persist(
    (...a) => ({
      ...createBearSlice(...a),
      ...createFishSlice(...a),
    }),
    { name: 'bound-store' },
  ),
)
```

### Create the following function: createSelectors

```typescript
import { StoreApi, UseBoundStore } from 'zustand'

type WithSelectors<S> = S extends { getState: () => infer T }
  ? S & { use: { [K in keyof T]: () => T[K] } }
  : never

const createSelectors = <S extends UseBoundStore<StoreApi<object>>>(
  _store: S,
) => {
  const store = _store as WithSelectors<typeof _store>
  store.use = {}
  for (const k of Object.keys(store.getState())) {
    ;(store.use as any)[k] = () => store((s) => s[k as keyof typeof s])
  }

  return store
}
```

```typescript
const useBearStore = createSelectors(useBearStoreBase)

// get the property
const bears = useBearStore.use.bears()

// get the action
const increment = useBearStore.use.increment()
```

Auto-typing

```typescript
import { create } from 'zustand'
import { combine } from 'zustand/middleware'

const useBearStore = create(
  combine({ bears: 0 }, (set) => ({
    increase: (by: number) => set((state) => ({ bears: state.bears + by })),
  })),
)
```

### Vanilla Store (createStore) VS Store (create)

### setState VS set

createStore -> useStore
