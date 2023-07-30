# Chapter 02: Local and Global States

React components come in a tree-like structure. For example:

```
           [1]
     [2]         [3]
  [4]   [5]   [6]   [7]
[8] [9]
```

As for local states, if one wants to create a state that is accessible by `[4] [5] [8] [9]`, then they will
place the state in `[2]` so that all of the child components can access the state. However, if `[3]` wants to
access the state stored in `[2]`, then the only way to do so is through _global states_

## When to Use Local States

### Pure Functions

**Pure function**: a function that only depends on the argument (i.e., returns the same value as long as the arguments are the same)

As states hold values outside arguments, functions that depend on a state becomes **impure** --> if `state` is used in a React component,
then the comonent is impure. A **contained** component is an impure function whose state is (1) local to the component and
(2) independent from the other components.

Pure function:

```jsx
const addOne = n => n + 1;
```

Impure function:

```jsx
let base = 1;
const addBase = n => n + base;
```

**Contained function:**

```jsx
const createContainer = () => {
  let base = 1;
  const addBase = n => n + base;
  const changeBase = b => {
    base = b;
  };
  return { addBase, changeBase };
};
const { addBase, changeBase } = createContainer();
```

In this case, the function is contained because all the state that the functions depend on (`base`) is a part of the container
and is (1) is in the same node in the React component tree and (2) will not affect other components.

> `let` will create a _block-scope_ variable, which will not be accessible outside the container. This will make the container
> a contained function because no other components can use the `base` variable

```jsx
const AddBase = ({ number }) => {
  const [base, changeBase] = useState(1);
  return <div>{number + base}</div>;
};
```

In this case, the `addBase` function is also contained.

### React Components and Props

```jsx
const Component = ({ number }) => {
  return <div>{number}</div>;
};
```

The React component is a JavaScript function, and the arguments are called props.

### Limitations of Local States

A global state is useful to control React component behavior from outside the component but makes the component less predictable.
Use local state as a primary mean and global state as a secondary mean.

## Using Local States

> Lifting state up = defining a state higher in the component tree; lifting content up = defining a content
> higher in the component tree

### Lifting state up

From two separate counters with separate states:

```jsx
const Component1 = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
    </div>
  );
};
const Component2 = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
    </div>
  );
};
```

To one single parent component that both components can access the state of:

```jsx
const Component1 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
    </div>
  );
};
const Component2 = ({ count, setCount }) => {
  // same as Component 1
};
const Parent = () => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1 count={count} setCount={setCount} />
      <Component2 count={count} setCount={setCount} />
    </>
  );
};
```

As the hook `count, setCount` is defined in `Parent`, both `Component1` and `Component2` could access them without needing to re-define them.

Now, the state is shared between the two components, and child components can use the parent component's state. Notably, once a
parent component re-renders, all the child components will re-render as well.

### Lifting Content Up

Lifting content up is recommended when a part of the code does not change, and thus do not require re-renders caused by changess in the parent
as the state of other child components change:

```jsx
const AdditionalInfo = () => {
  return <p>Some information</p>;
};

const Component1 = ({ count, setCount, additionalInfo }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
      {additionalInfo}
    </div>
  );
};

const Parent = ({ additionalInfo }) => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1
        count={count}
        setCount={setCount}
        additionalInfo={additionalInfo}
      />
      <Component2 count={count} setCount={setCount} />
    </>
  );
};

const GrandParent = () => {
  return <Parent additionalInfo={<AdditionalInfo />} />;
};
```

> Changing `Component1` will trigger `Parent` to be re-rendered because it invokes the `setCount` function to change
> the `count` in `Parent`'s state.

The code above prevents re-render of the `AdditionalInfo` component,
which is passed as an argument down from the `Grandparent` (which will _not_ re-render) into `Parent` (which _will_ re-render
`Component1` and `Component2` as either changes but not `AdditionalInfo`), effectively reducing an unnecessary re-render.

Another way to phrase this would be:

```jsx
const AdditionalInfo = () => {
  return <p>Some information</p>;
};

const Component1 = ({ count, setCount, children }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
      {children}
    </div>
  );
};

const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1 count={count} setCount={setCount}>
        {children}
      </Component1>
      <Component2 count={count} setCount={setCount} />
    </>
  );
};

const GrandParent = () => {
  return (
    <Parent>
      <AdditionalInfo />
    </Parent>
  );
};
```

`children` is a special prop name that represents all the nested children elements within the `Parent` component wrapper.
The fundamental concept is still the same, which is to pass `AdditionalInfo` as if it's an argument up to the parent
components; but instead of putting it as an actual argument (like in the previous example), this one is done through `children` props.

## Global State

If a state belongs to a single component and is encapsulated by the component, it is a local state. Otherwise, it is a global state.

Global state can be used when:

1. passing a prop is not desirable
2. there is already a state outside of react

### Undesirably Passing Props

Passing props is not desirable when there are too many intermediaries:

```jsx
const Component1 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
    </div>
  );
};
const Parent = ({ count, setCount }) => {
  return <Component1 count={count} setCount={setCount} />;
};
const GrandParent = ({ count, setCount }) => {
  return <Parent count={count} setCount={setCount} />;
};
const Root = () => {
  const [count, setCount] = useState(0);
  return <GrandParent count={count} setCount={setCount} />;
};
```

Where passing props through multi-level intermediate components might not result in a good developer experience as it could seem like extra work.
Furthermore, the intermediate components will also re-render when the state is updated, which could impact performance.

A similar example using a global state will look like:

```jsx
const Component1 = () => {
  // useGlobalCountState is a pseudo hook
  const [count, setCount] = useGlobalCountState();
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}></button>
    </div>
  );
};
const Parent = () => {
  return <Component1 />;
};
const GrandParent = () => {
  return <Parent />;
};
const Root = () => {
  return <GrandParent />;
};
```

Which saves a lot of code and improves readability; in addition, the parent components will not re-render when the child component is changed.

### Already Having a State

```jsx
const globalState = {
  authInfo: { name: 'React' },
};
const Component1 = () => {
  // useGlobalState is a pseudo hook
  const { authInfo } = useGlobalState();
  return <div>{authInfo.name}</div>;
};
```

Since `globalState` already exists, one needs `useGlobalState` to link `Component1` with `globalState` so that the former can receive `authInfo`.
