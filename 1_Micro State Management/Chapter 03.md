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
-

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
