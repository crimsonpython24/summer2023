# Chapter 4: Sharing Module State with Subscription

Context: not designed for singleton pattern (only allows one instantiation of a class) but to avoid the singleton pattern and provide
different values for different subtrees (context providers).

For a singleton-like global state, it's more sensible to use a module state.

> The Module State is a variable defined globally or within the scope of a file

## The Module State

```jsx
let count = 0;
```

This is a module state.

```jsx
let state = {
  count: 0,
};
```

This is an object state. Most commonly in react, it's better to have an access and set function wrapped together with the value. In that case,
one can create a container:

```jsx
export const createContainer = initialState => {
  let state = initialState;
  const getState = () => state;
  const setState = nextState => {
    state = typeof nextState === 'function' ? nextState(state) : nextState;
  };
  return { getState, setState };
};
```

The function can be used as follows:

```jsx
import { createContainer } from '...';

const { getState, setState } = createContainer({
  count: 0,
});

setState(prevState => ({
  ...prevState,
  count: prevState.count + 1,
}));
```

## Using a Module State as a Global State

Although React Context can provide different values for different subtrees, using Context for a singleton global state does not use the full
capability of Context. If one only needs a global state for an entire tree, a module state might fit better; however, one needs to handle
re-rendering themselves.

For instance:

```jsx
let count = 0;

const Component1 = () => {
  const inc = () => {
    count += 1;
  };
  return (
    <div>
      {count} <button onClick={inc}>+1</button>
    </div>
  );
};

const Component2 = () => {
  // ...
};
```

Without a context provider, this will **not** re-render by itself. The components will only re-render when the component's own button is clicked.

(The following is a working counter from the previous section)

```jsx
const CountStateContext = createContext({
  count: 0,
  setCount: () => {},
});

const Component1 = () => {
  const { count, setCount } = useContext(CountStateContext);
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </div>
  );
};

const Component2 = () => {
  // ...
};

const Parent = () => {
  return (
    <>
      <Component1 />
      <Component2 />
    </>
  );
};

export default function App() {
  const [count, setCount] = useState(0);
  return (
    <CountStateContext.Provider value={{ count, setCount }}>
      <Parent />
    </CountStateContext.Provider>
  );
}
```

## Basic Subscription

Subscription is a way to get notified of things such as updates. In the module state implementation, `store` will hold the `state` value and
the `subscribe` method, in addition to `getState` and `setState`. `createStore` is used to initialize the `store` with a value:

```jsx
const createStore = initialState => {
  let state = initialState;
  const callbacks = new Set();
  const getState = () => state;
  const setState = nextState => {
    state = typeof nextState === 'function' ? nextState(state) : nextState; // [1]
    callbacks.forEach(callback => callback()); // [2]
  };
  const subscribe = callback => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback); // [3]
    };
  };
  return { getState, setState, subscribe };
};
```

In `[1]`, the state will be altered depending on the type of parameter `setState` is called. If `setState` is called with a function in the brackets,
then the `state` variable will be updated with a function; otherwise, the `state` variable will just be replaced with another value (`nextState`) if
the parameter is not a function.

For this code segment, the list of callbacks are stored in the `callbacks` set; when `setState` is invoked, each of the callback functions that are
"watched" will be called in `[2]`. I.e., the functions that are watched in the `callbacks` set that are added through `subscribe` will be executed
when `setState` is called because the `forEach` loop will update all the watched components that are added via the `subscribe` method.

The purpose of `[3]` is to clean off the side effects before the corresponding component leaves the screen (such as through a page reload), where the
clean-up funciton runs before the component is removed from the UI to prevent memory leaks. If a component is rendered multiple times, the previous
effects will be cleaned off before the next effect.

References:
[Hooks API](https://legacy.reactjs.org/docs/hooks-reference.html)
[Hooks Effect](https://legacy.reactjs.org/docs/hooks-effect.html)

The `createStore` const above can be used as:

```jsx
const store = createStore({ count: 0 });
console.log(store.getState());
store.setState({ count: 1 });
store.subscribe(...);
```

Then one can define a hook that returns a tuple of the `store` state value and its update function:

```jsx
const useStore = store => {
  const [state, setState] = useState(store.getState());
  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });
    setState(store.getState()); // [1]
    return unsubscribe;
  }, [store]);
  return [state, store.setState];
};
```

`[1]` is to cover the edge case where `useEffect` may get delayed and there's a chance that `store` already has a new state.

```jsx
const Component1 = () => {
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

const Component2 = () => {
  // ...
};

const App = () => (
  <>
    <Component1 />
    <Component2 />
  </>
);
```

Each time the component is executed, the `useStore` hook will be called first, and then `callbacks.add(callback)` and `setState(store.getState()) // [1]`. This
will be loaded each time the page is refreshed. If somehow the component is rendered twice (e.g., `strictMode` is on), then the two functions above will be run
again once the side effects are eliminated (i.e., `callbacks.delete` --> `callbacks.add` and `setState`).

Next, every time the function is invoked, the `setState(store.getState());` inside the unsubscribe function will be called, as the `useEffect` hook will be watching
`store`, and will fire if `store` changes. After the component has unsubscribed, then the hook itself will fire (given that the component uses `setState`, which
comes from `useStore`). This step will occur each time the "+1" button is clicked.

## Working with a Selector

Using a selector will allow one to return the only part of a state component that is in interest. With the same `createStore` function as
above, the object now consists of two counters:

```jsx
const store = createStore({ count1: 0, count2: 0 });
```

```jsx
const useStoreSelector = (store, selector) => {
  const [state, setState] = useState(() => selector(store.getState()));
  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(selector(store.getState()));
    });
    setState(selector(store.getState()));
    return unsubscribe;
  }, [store, selector]);
  return state;
};
```

```jsx
const Component1 = () => {
  const state = useStoreSelector(
    store,
    useCallback(state => state.count1, [])
  );
  const inc = () => {
    store.setState(prev => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };
  return (
    <div>
      count1: {state} <button onClick={inc}>+1</button>
    </div>
  );
};

const selectCount2 = state => state.count2;

const Component2 = () => {
  const state = useStoreSelector(store, selectCount2);
  const inc = () => {
    store.setState(prev => ({
      ...prev,
      count2: prev.count2 + 1,
    }));
  };
  return (
    <div>
      count2: {state} <button onClick={inc}>+1</button>
    </div>
  );
};
```

```jsx
const App = () => (
  <>
    <Component1 />
    <Component1 />
    <Component2 />
    <Component2 />
  </>
);
```

`useSubscription` is a hook that is implemented by React:

```jsx
const useStoreSelector = (store, selector) =>
  useSubscription(
    useMemo(
      () => ({
        getCurrentValue: () => selector(store.getState()),
        subscribe: store.subscribe,
      }),
      [store, selector]
    )
  );
```

```jsx
const Component1 = () => {
  const state = useSubscription(
    useMemo(
      () => ({
        getCurrentValue: () => store.getState().count1,
        subscribe: store.subscribe,
      }),
      []
    )
  );
  const inc = () => {
    store.setState(prev => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };
  return (
    <div>
      count1: {state} <button onClick={inc}>+1</button>
    </div>
  );
};
```
