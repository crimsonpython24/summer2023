# Chapter 5: Sharing Component State with Context and Subscription

## Limitations of the Module State

The module state, as it resides outside React components, is a singleton adn there cannot bve different states for different component
trees or subtrees. Since the module state is a simple value, it cannot provide different values for different subtrees.

For instance, using this counter example:

```jsx
const Counter = () => {
  const [state, setState] = useStore(store);
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      {state.count} <button onClick={inc}>+1</button>
    </div>
  );
};
const Component = () => (
  <>
    <Counter />
    <Counter />
  </>
);

const Counter2 = () => {
  const [state, setState] = useStore(store2);
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      {state.count} <button onClick={inc}>+1</button>
    </div>
  );
};
const Component2 = () => (
  <>
    <Counter2 />
    <Counter2 />
  </>
);
```

Every time a new `Counter` is created, every time a new `Counter` must be defined from scratch. This lack of reusability
is because that the module state is defined outside of React so it's not possible for them to be reused.

## When to Use Context

Context can be used when one needs to provide a different value for the subtree of the entire component tree (note that
one can overwrite context provider values by nesting them); otherwise, if this is not needed, then one can simply use
the default value in the provider.

As one should only use one provider at the root, that use case can be covered by module state with Subscription. I.e.,
the module state covers the use case with one Context provider at the root, and Context is only required if one needs to
provide different values for different subtrees.

## Implementing Context with Subscription

```jsx
const createStore = initialstate => {
  let state = initialState;
  const callbacks = new Set();
  const getState = () => state;
  const setState = nextState => {
    state = typeof nextState === 'function' ? nextState(state) : nextState;
    callbacks.forEach(callback => callback());
  };
  const subscribe = callback => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };

  return { getState, setState, subscribe };
};
```

```jsx
type State = { count: number, text?: string };
const StoreContext =
  createContext <
  Store <
  State >> (createStore < State > { count: 0, text: 'hello' });
```

```jsx
const StoreProvider = (initialState, children) => {
  const storeRef = useRef();
  if (!storeRef.current) {
    storeRef.current = createStore(initialState);
  }
  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
};
```

```jsx
const useSelector = selector => {
  const store = useContext(StoreContext);
  return useSubscription(
    useMemo(
      () => ({
        getCurrentValue: () => selector(store.getState()),
        subscribe: store.subscribe,
      }),
      [store, selector]
    )
  );
};
```

```jsx
const useSetState = () => {
  const store = useContext(StoreContext);
  return store.setState;
};
```

```jsx
const selectCount = state => state.count;
const Component = () => {
  const count = useSelector(selectCount);
  const setState = useSetState();
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  );
};
```

```jsx
const App = () => (
  <>
    <h1>Using default store</h1>
    <Component />
    <Component />
    <StoreProvider initialState={{ count: 10 }}>
      <h1>Using store provider</h1>
      <Component />
      <Component />
      <StoreProvider initialState={{ count: 20 }}>
        <h1>Using inner store provider</h1>
        <Component />
        <Component />
      </StoreProvider>
    </StoreProvider>
  </>
);
```
