# Chapter 1: Basics

## Definitions

| Term                   | Explanation                                                                                                                                          |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| React hooks            | Primitive, reusable hooks that can make state mangement lightweight (micro)                                                                          |
| Micro state management | Purpose-orientated state management used with specific patterns (monolithic: more general, e.g., through Redux)                                      |
| State                  | Any data that represents the UI and can change over time and can be used for specific purposes (form state, server cache state, navigation state...) |

## Hook Basics

- `useState`: basic function to create a local state
- `useReducer`: can also create a local state and may replace `useState` in certain cases
- `useEffect`: run logic outside of the React render process

Hooks allow logic to be extracted, using a "counter" example:

```jsx
const useCount = () => {
  const [count, setCount] = useState(0); // [2]
  return [count, setCount];
};

const Component = () => {
  const [count, setCount] = useCount(); // [1]
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </div>
  );
};
```

[1] Improves readability: as opposed to using `useState` inline, a defined name labels functions more clearly.
[2] As `useCount` is extracted from `Component`, one can update the `useCount` hook without breaking the `Component` component.

Extraction logic is evident as follows:

```jsx
const useCount = () => {
  const [count, setCount] = useState(0);
  const inc = () => setCount(c => c + 1);
  return [count, inc];
};

// ...
<button onClick={() => setCount()}>+1</button>;
```

### More Advanced Cases

- **Suspense for Data Fetching**: allows one to code components without worrying about `async`
- **Concurrent rendering**: method to split process into chunks and prevent blocking the CPU for a prolonged period of time

### Don't-s

Do not directly mutate (i.e., change/set) a `state` or a `ref` object, as doing so may lead to unexpected behavior (incomplete/failed/multiple re-renders);
as hook functions have to be invoked multiple times, the functions should behave consistently

## Local vs Global States

_Local State_: state that are defined in a component and used in a component tree:

```jsx
const Component = () => {
  const [state, setState] = useState();
  return (
    <div>
      {JSON.stringify(state)}
      <Child state={state} setState={setState} />
    </div>
  );
};

const Child = ({ state, setState }) => {
  const setFoo = () => setState(prev => ({ ...prev, foo: 'foo' }));
  return (
    <div>
      {JSON.stringify(state)}
      <button onClick={setFoo}>Set Foo</button>
    </div>
  );
};
```

In this example, the `state` from `Component` is passed down to the `Child` component. When the button is clicked and triggers `setFoo`, the
two `<div>` elements in `Component` and `Child` will display `{"foo":"foo"}` (the three dots deconstruct an array/object into separate variables,
in this case to concatenate the dictionary with the new key-value pair)

_Global State_: state that is consumed in multiple components, often not in the same component tree and far apart in an app

- as React is based on the Component model, implementing global states in React is not an easy task
  - in an component model, a component should be isolated and reusable (**locality**)
  - as React does not have a direct solution, there are many proposed solutions with their own pros and cons

Once a global state is defined, it can be used anywhere in the application:

```jsx
const Component1 = () => {
  const [state, setState] = useGlobalState();
  return <div>{JSON.stringify(state)}</div>;
};
```

without having to re-define `useGlobalState` (this is not an official function, but a hypothetical global state)

## `useState`

### Updating the State Value with a Value

```jsx
const Component = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(1)}>Set Count to 1</button>
    </div>
  );
};
```

When the button is clicked again, `setCount(1)` is triggered, but the function will "bail out" because the value is the same. **Bailout** in React
refers to avoid triggering re-renders if something's value doesn't change.

Now, suppose the counter is a part of a dictionary-like state:

```jsx
const Component = () => {
  const [state, setState] = useState({ count: 0 });
  return (
    <div>
      {state.count}
      <button onClick={() => setState({ count: 1 })}>Set Count to 1</button>
    </div>
  );
};
```

The object _will_ re-render because it creates a new object that has `{count: 1}`, which is different from the previous object even if the value is the same
(goes back to all the type mess JavaScript has, but essentially, dictionary is considered an object with a unique instance)

### Updating the State Value with a Function

Sometimes, updating states with a value wouldn't work. For example:

```jsx
const Component = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>
        Set Count to {count + 1}
      </button>
    </div>
  );
};
```

If clicked repeatedly fast enough, the function above would only update once. This is because the function depends on the _displayed_ value (count). This will
result in the function depending on the value that is written on the button. I.e., if the button's `count` (displayed value) does not update in time, `count + 1` will be the
same for consecutive clicks, such as when clicks are repeated fast enough.

The function below is based on the "previous value" (`c`):

```jsx
const Component = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
    </div>
  );
};
```

As this function is based on the previous value, the counter will increment correctly at each click, even if the button's value may not be updated in time. `c` here refers
to the previous value immediately before the function is clicked.

Notably, bailout will also work with function updates, such as invoking `setCount((c) => c)`

### Lazy Init

```jsx
const init = () => 0;
const Component = () => {
  const [count, setCount] = useState(init);
  return <span></span>;
};
```

If the inital computation of a state is heavy, extracting it into a seaparate function will only cause the value to be evaluated in the first render,
but not immediately once the site is loaded (obviously 0 is not computation-intensive, but still).

I.e., this component will not be evaluated before calling `useState` but invoked just once on `mount`.

## `useReducer`

### Example

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'SET_TEXT':
      return { ...state, text: action.text };
    default:
      throw new Error('unknown action type');
  }
};

const Component = () => {
  const [state, dispatch] = useReducer(reducer, { count: 0, text: 'hi' });
  return (
    <div>
      {state.count}
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>
        Increment count
      </button>
      <input
        value={state.text}
        onChange={e => dispatch({ type: 'SET_TEXT', text: e.target.value })}
      />
    </div>
  );
};
```

### Bailout

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    // ...
    case 'SET_TEXT':
      if (!action.text) {
        return state; // bail out
      }
      return { ...state, text: action.text };
    default:
      throw new Error('unknown action type');
  }
};
```

This is a bailout example because it avoids re-rendering the object if its value doesn't change.

### Lazy Init

`useReducer` has two necessary and one optional parameter:

1. reducer function
2. initial state
3. lazy initialization

```jsx
const init = count => ({ count, text: 'hi' });

const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'SET_TEXT':
      return { ...state, text: action.text };
    default:
      throw new Error('unknown action type');
  }
};

const Component = () => {
  const [state, dispatch] = useReducer(reducer, 0, init);
  // ...
};
```

Note that the `init` function takes two parameters in this case, the first being the state name itself and the second being the initial state,
whereas the lazy initialization for `useState` only needs one parameter being the intial state.

## `useState` vs `useReducer`

Note that `useState` is implemented with `useReducer` inside React (as of now):

```jsx
const reducer = (prev, action) =>
  typeof action === 'function' ? action(prev) : prev;
const useState = initialState => useReducer(reducer, initialState);
```

However, the opposite does not hold true. One difference being that `reducer` and `init` can be defined outside hooks or components with `useReducer`
but is not possible with `useState`:

```jsx
const init = count => ({ count });
const reducer = (prev, delta) => prev + delta;
const ComponentWithUseReducer = ({ initialCount }) => {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  return (
    // ...
  );
};
const ComponentWithUseState = ({ initialCount }) => {
  const [state, setState] = useState(() => init(initialCount));
  const dispatch = delta => setState(prev => reducer(prev, delta));
  return (
    // ...
  );
};
```
