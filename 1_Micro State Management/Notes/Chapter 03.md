# Chapter 3: Sharing Component State with Context

## Context

**Context**: passing data from component to component instead of using props.

- If context is used in coordination with `useState`, one can create a custom hook for a global state
- However, one known limitation is that all Context comsuners re-render upon updates, which could lead to _extra_ re-renders --> recommended to
  split a gglobal state into pieces

### Prop Drilling

As illustrated in Chapter 2, **Prop Drilling** is a common pattern used to pass props from parent to children repetitively.

```jsx
const App = () => {
  const [count, setCount] = useState(0);
  return <Parent count={count} setCount={setCount} />;
};

const Parent = ({ count, setCount }) => (
  <>
    <Component1 count={count} setCount={setCount} />
    <Component2 count={count} setCount={setCount} />
  </>
);

const Component1 = ({ count, setCount }) => (
  <div>
    {count}
    <button onClick={() => setCount(c => c + 1)}>+1</button>
  </div>
);

const Component2 = ({ count, setCount }) => (
  // ...
);
```

However, it can be observed that `Parent` does not need to know about the `count` state given that it's only an intermediary between `App` and
`Component1`/`Component2`. Stopping `Parent` from re-rendering can also prevent unnecessary re-rendering in other child components under it.
Which, in addition, also becomes complicated and difficult to manage in large-scale applications.

### `useContext`

Context is a method to pass value from a parent to a child component without using props.

- There can be multiple providers that provide different values.
- Providers can be nested, and a **consumer** component (a component that uses the `useContext` hook) will pick the closest provider in the
  component tree to get the context value.

```jsx
const ColorContext = createContext('black');
```

This line creates a context.

```jsx
const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};
```

`Component` will have no idea what the color is, the only dependency being the Context.

```jsx
const App = () => (
  <>
    <Component />
    <ColorContext.Provider value='red'>
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value='green'>
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value='blue'>
      <Component />
      <ColorContext.Provider value='skyblue'>
        <Component />
      </ColorContext.Provider>
    </ColorContext.Provider>
  </>
);
```

The first component will show `black`, the second and third `red` and `green`, and the last `skyblue`.

A vital capability of React Context is the ability to reuse the consumer component; however, if this is not needed, then subscription without
context might be a more suitable method (chapter 4).

## Using `useState` with `useContext`

> Idea: pass the `state` value and update function in Context instead of props. In this way, the `Parent` component does not need to take props,
> while `Component1` and `Component2` uses `useContext` to get the state.

```jsx
const CountStateContext = createContext({
  count: 0,
  setCount: () => {},
});
```

This context, much like hooks, represents a value and the updater function. `0` and `() => {}` are only used to infer types (static value
and function) in TypeScript. Given that `count` will be a state rather than a static value, the default values hold little significance.

Using the Context in the application:

```jsx
const App = () => {
  const [count, setCount] = useState(0);
  return (
    <CountStateContext.Provider value={{ count, setCount }}>
      <Parent />
    </CountStateContext.Provider>
  );
};
```

As `count` and `setCount` are passed down as arguments to the Context, all the child components could access it through the `useContext` hook
that will grant them access to the same `[count, setCount]` as in `App`.

Similar to the example in the previous section, we must provide an initial value to the context provider. This is why the default values does
little besides hinting at the data types each field holds.

```jsx
const Parent = () => (
  <>
    <Component1 />
    <Component2 />
  </>
);
```

In this case, the parent component does not know about the existence of the `count` state, while the components inside `Parent` can still use
the `count` state through the Context:

```jsx
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
```

## Understanding Context

When a provider has a new Context value, all the consumers will re-render upon receiving the new value (i.e., the value in the provider is **propogated**
to the consumers)

The following test will consist of four components:

```jsx
const Parent = () => (
  <>
    <DummyComponent />
    <MemoedDummyComponent />
    <ColorComponent />
    <MemoedColorComponent />
  </>
);
```

```jsx
const App = () => {
  const [color, setColor] = useState('red');
  return (
    <ColorContext.Provider value={color}>
      <input value={color} onChange={e => setColor(e.target.value)} />
      <Parent />
    </ColorContext.Provider>
  );
};
```

```jsx
const ColorComponent = () => {
  const color = useContext(ColorContext);
  const renderCount = useRef(1);
  useEffect(() => {
    renderCount.current += 1;
  });
  return (
    <div style={{ color }}>
      Hello {color} (renders: {renderCount.current})
    </div>
  );
};
```

`renderCount.current` displays the render count number.

```jsx
const MemoedColorComponent = memo(ColorComponent);
```

```jsx
const DummyComponent = () => {
  const renderCount = useRef(1);
  useEffect(() => {
    renderCount.current += 1;
  });
  return <div>Dummy (renders: {renderCount.current})</div>;
};
```

```jsx
const MemoedDummyComponent = memo(DummyComponent);
```

Once the text input changes, the `App` component renders because of `useState`. `ColorContext.Provider` will get a new value, and since `Parent` is a child
of `App`, `Parent` will also re-render.

However, each of the component will follow different behaviors:

| Component              | Behavior       | Explanation                                                             |
| ---------------------- | -------------- | ----------------------------------------------------------------------- |
| `DummyComponent`       | Renders        | A child of `Parent`, which is a child of `App`                          |
| `MemoedDummyComponent` | Doesn't render | Doesn't re-render because the `memo` function prevents so               |
| `ColorComponent`       | Renders        | Renders because (1) the parent renders, and (2) the context changes     |
| `MemoedColorComponent` | Renders        | Renders only because the context changes, but not because of the parent |

Note that the `memo` function creates a memoized (an optimization technique used to store expensive calculations) component from the base component; `memo`
does not stop the internal Context consumer from re-rendering because components will display outdated Context values otherwise.

### Limitations of `useContext` for Objects

If there is an object being passed as a context, for example:

```jsx
const CountContext = createContext({ count1: 0, count2: 0 });
```

And then have a component utilizing each part of the context:

```jsx
const Counter1 = () => {
  const { count1 } = useContext(CountContext);
  const renderCount = useRef(1);
  useEffect(() => {
    renderCount.current += 1;
  });
  return (
    <div>
      Count1: {count1} (renders: {renderCount.current})
    </div>
  );
};
const Counter2 = () => {
  const { count2 } = useContext(CountContext);
  const renderCount = useRef(1);
  useEffect(() => {
    renderCount.current += 1;
  });
  return (
    <div>
      Count2: {count2} (renders: {renderCount.current})
    </div>
  );
};
```

When either `Counter1` or `Counter2` is triggered, both counters will be re-rendered, even if there will always be a counter whose value isn't changed.

Although extra re-renders should be technically avoided, they are fine as long as they do not heavily impact performance, and overengineering to resolve the
few extra re-renders may lead to more problems and not worth the effort.

## Creating a Context for a Global State

### Creating Small State Pieces

To prevent multiple re-renders, one could define two contexts:

```jsx
type CountContextType = [number, Dispatch<SetStateAction<number>>];
const Count1Context = createContext < CountContextType > [0, () => {}];
const Count2Context = createContext < CountContextType > [0, () => {}];
```

So that in each counter, it will only invoke the corresponding context:

```jsx
const Counter1 = () => {
  const [count1, setCount1] = useContext(Count1Context);
  return (
    <div>
      Count1: {count1}
      <button onClick={() => setCount1(c => c + 1)}>+1</button>
    </div>
  );
};
const Counter2 = () => {
  // ...
};
```

```jsx
const Count1Provider = ({ children }: { children: ReactNode }) => {
  const [count1, setCount1] = useState(0);
  return (
    <Count1Context.Provider value={[count1, setCount1]}>
      {children}
    </Count1Context.Provider>
  );
};

const Count2Provider = ({ children }: { children: ReactNode }) => {
  // ...
};
```

This final code segment is synonymous to the earlier code segment (in `useContext` without `useState`):

```jsx
<ColorContext.Provider value='red'>
  <Component />
</ColorContext.Provider>
```

As in the providers give a default value to the context. However, as `useState` and `useContext` is combined in this case, the providers must contain
states rather than primitive values.

And the final steps to do are to wrap the component within the context providers:

```jsx
const Parent = () => (
  <div>
    <Counter1 />
    <Counter2 />
  </div>
);

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);
```

### Propagating with Multiple Contexts

> Create a single state and use multiple Contexts to distribute the state in pieces

```jsx
type Action = { type: 'INC1' } | { type: 'INC2' };
const Count1Context = createContext < number > 0;
const Count2Context = createContext < number > 0;
const DispatchContext = createContext < Dispatch < Action >> (() => {});
```

If there are more counts, then there will be more contexts, but still only one `DispatchContext`

```jsx
const Counter1 = () => {
  const count1 = useContext(Count1Context);
  const dispatch = useContext(DispatchContext);
  return (
    <div>
      Count1: {count1}
      <button onClick={() => dispatch({ type: 'INC1' })}>+1</button>
    </div>
  );
};
const Counter2 = () => {
  // ...
};
```

`Counter1` reads `count1` from `Count1Context`, but both counters will read from the same dispatch context.

The parent is still the same:

```jsx
const Parent = () => (
  <>
    <Counter1 />
    <Counter2 />
  </>
);
```

```jsx
const Provider = ({ children }: { children: ReactNode }) => {
  const [state, dispatch] = useReducer(
    (prev: { count1: number, count2: number }, action: Action) => {
      if (action.type === 'INC1') {
        return { ...prev, count1: prev.count1 + 1 };
      }
      if (action.type === 'INC2') {
        return { ...prev, count2: prev.count2 + 1 };
      }
      throw new Error('no matching action');
    },
    {
      count1: 0,
      count2: 0,
    }
  );
  return (
    <DispatchContext.Provider value={dispatch}>
      <Count1Context.Provider value={state.count1}>
        <Count2Context.Provider value={state.count2}>
          {children}
        </Count2Context.Provider>
      </Count1Context.Provider>
    </DispatchContext.Provider>
  );
};
```

The `useReducer` hook serves a similar purpose to `useState`, but gives a more organized approach for updating non-primitive values. In this case, as
there is an object whose keys should be updated independently, `useReducer` will be a better hook to initialize the state.

There are two required arguments for a reducer:

1. `reducer`: a pure function that takes a state, and returns a new state based on the previous state and an action
2. `initialState`: an initial state value like those in `useState`

And the application wrapper is simplified:

```jsx
const App = () => (
  <Provider>
    <Parent />
  </Provider>
);
```

## Best Practices

### Creating Custom Hooks and Providers

Creating a custom hook and provider will allow one to hide Contexts, as custom cooks can directly access Context values and provider components.

The first thing to do is to create a Context, which is `null` in this example; this also signals that the default value cannot be used, and that values
should always be accessed through a provider.

```jsx
type CountContextType = [number, Dispatch<SetStateAction<number>>];
const Count1Context = (createContext < CountContextType) | (null > null);
```

The `Count1Provider` then creates a state with `useState` and passes it to the Context provider.

```jsx
export const Count1Provider = ({ children }: { children: ReactNode }) => (
  <Count1Context.Provider value={useState(0)}>
    {children}
  </Count1Context.Provider>
);

// Note that value={useState(0)} is an abbreviation for:
const [count, setCount] = useState(0);
return <Count1Context.Provider value={[count, setCount]}>
```

The `useCount1` hook is then defined to return a value from `Count1Context`:

```jsx
export const useCount1 = () => {
  const value = useContext(Count1Context);
  if (value === null) throw new Error('Provider missing');
  return value;
};
```

Finally, the `Counter1` component is created to use the `count1` state using the `useCount1()` hook that is defined:

```jsx
const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Count1: {count1}
      <button onClick={() => setCount1(c => c + 1)}>+1</button>
    </div>
  );
};

const Parent = () => (
  <div>
    <Counter1 />
    <Counter2 />
  </div>
);

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);
```

### Factory Pattern with a Custom Hook (TypeScript-only)

As creating a custom hook and provider component can be repetitive, one can write a function that does the task.

This example shows `createStateContext`: it takes a `useValue` custom hook; if one uses `useState`, the context returns a tuple of the `state` value
and the `setState` function (i.e., `createStateContext` returns a tuple of a provider and a custom hook to get the state). This Context also provides
an optional choice to pass an `initialValue` prop into `useValue` so that one can set an initial value at runtime instead of at creation.

```jsx
const createStateContext = () => {
  const StateContext = createContext(null);
  const StateProvider = ({ initialValue, children }) => (
    <StateContext.Provider value={useValue(initialValue)}>
      {children}
    </StateContext.Provider>
  );
  const useContextState = () => {
    const value = useContext(StateContext);
    if (value === null) throw new Error('Provider missing');
    return value;
  };
  return [StateProvider, useContextState];
};
```

```jsx
const useNumberState = init => useState(init || 0);
```

This custom hook takes an optional `init` parameter, which will be used with `useState` and `init`:

```jsx
const [Count1Provider, useCount1] = createStateContext(useNumberState);
const [Count2Provider, useCount2] = createStateContext(useNumberState);
```

With this code, one can avoid repetitions while defining custom hooks. Finally, the Counter can be defined as such:

```jsx
const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Best practices for using Context 69 Count1: {count1}
      <button onClick={() => setCount1(c => c + 1)}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  // ...
};

const Parent = () => (
  <div>
    <Counter1 />
    <Counter2 />
  </div>
);

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);
```

One can also make a custom hook with `useReducer`:

```jsx
const useMyState = (initialState = { count1: 0, count2: 0 }) => {
  const [state, setState] = useState(initialState);
  useEffect(() => {
    console.log('updated', state);
  });
  const inc1 = useCallback(() => {
    setState(prev => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  }, []);
  const inc2 = useCallback(() => {
    setState(prev => ({
      ...prev,
      count2: prev.count2 + 1,
    }));
  }, []);
  return [state, { inc1, inc2 }];
};
```

This more complicated version of the `useNumberState` allows for custom action functions and logging in the console through `useEffect`.

This following rewrite implements a stricter type-checking:

```tsx
const createStateContext = <Value, State>(
  useValue: (init?: Value) => State
) => {
  const StateContext = createContext<State | null>(null);
  const StateProvider = ({
    initialValue,
    children,
  }: {
    initialValue?: Value;
    children?: ReactNode;
  }) => (
    <StateContext.Provider value={useValue(initialValue)}>
      {children}
    </StateContext.Provider>
  );
  const useContextState = () => {
    const value = useContext(StateContext);
    if (value === null) {
      throw new Error('Provider missing');
    }
    return value;
  };
  return [StateProvider, useContextState] as const;
};
const useNumberState = (init?: number) => useState(init || 0);
```

> The code snippets above is in TS, as `useValue` is a CoreTS hook. There are also type-checking involved in this stricter definition.

### Avoiding Provider Nesting

From this:

```jsx
const [Count1Provider, useCount1] = createStateContext(useNumberState);
const [Count2Provider, useCount2] = createStateContext(useNumberState);
const [Count3Provider, useCount3] = createStateContext(useNumberState);
const [Count4Provider, useCount4] = createStateContext(useNumberState);
const [Count5Provider, useCount5] = createStateContext(useNumberState);

const App = () => (
  <Count1Provider initialValue={10}>
    <Count2Provider initialValue={20}>
      <Count3Provider initialValue={30}>
        <Count4Provider initialValue={40}>
          <Count5Provider initialValue={50}>
            <Parent />
          </Count5Provider>
        </Count4Provider>
      </Count3Provider>
    </Count2Provider>
  </Count1Provider>
);
```

Into this:

```jsx
const App = () => {
  const providers = [
    [Count1Provider, { initialValue: 10 }],
    [Count2Provider, { initialValue: 20 }],
    [Count3Provider, { initialValue: 30 }],
    [Count4Provider, { initialValue: 40 }],
    [Count5Provider, { initialValue: 50 }],
  ];
  return providers.reduceRight(
    (children, [Component, props]) => createElement(Component, props, children),
    <Parent />
  );
};
```
