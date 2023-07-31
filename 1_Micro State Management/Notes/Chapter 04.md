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
    state = typeof nextState === 'function' ? nextState(state) : nextState; // [2]
    callbacks.forEach(callback => callback()); // [1]
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

For this code segment, the list of callbacks are stored in the `callbacks` set; when `setState` is invoked, each of the callback functions that are
"watched" will be called in `[1]`. I.e., the functions that are watched in the `callbacks` set will be executed when `setState` is called because
`setState` should update all the watched components that are added through the `subscribe` method.

In `[2]`, the state will be altered depending on the type of parameter `setState` is called. If `setState` is called with a function in the brackets,
then the `state` variable will be updated with a function; otherwise, the `state` variable will just be replaced with another value if the parameter
is not a function.

The expectation for `[3]` is that it will return an unsubscribe function. It is called when the snippet loads (so it will load four times); when
this function is used without the `unsubscribe` part:

```jsx
store.subscribe(() => {
  setState(selector(store.getState()));
});
```

Then no values will be deleted, and the return statement would not be invoked. However, if `unsubscribe` is called as a return value, then the code
segment will be executed. Think of `[3]` as, when called with a return value, a temporary method that will push the function callback into the `callbacks`
set and removed once the component has finished loading. When the action is dispatched after the state was updated, the store subscription will become
active, and remember to unsubscribe.

Which can be used as:

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
