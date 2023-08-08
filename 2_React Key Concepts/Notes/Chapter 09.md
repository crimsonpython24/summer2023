# Chapter 9

**Component functions** are executed whenever their internal state changes or their parent function is executed again (any component functions referenced in the parent's JSX code are invoked on re-renders).

## Component Updates

Consider the following code segment:

```jsx
function NestedChild() {
  console.log('<NestedChild /> is called.');
  return (
    <p id='nested-child'>A component, deeply nested into the component tree.</p>
  );
}
function Child() {
  console.log('<Child /> is called.');
  return (
    <div id='child'>
      <p>
        another A component, rendered inside another component, containing yet
        component.
      </p>
      <NestedChild />
    </div>
  );
}
function Parent() {
  console.log('<Parent /> is called.');
  const [counter, setCounter] = useState(0);
  function incCounterHandler() {
    setCounter(prevCounter => prevCounter + 1);
  }
  return (
    <div id='parent'>
      <p>A component, nested into App, containing another component (Child).</p>
      <p>Counter: {counter}</p>
      <button onClick={incCounterHandler}>Increment</button>
      <Child />
    </div>
  );
}

function App() {
  return <Parent />;
}
```

The code segment above will re-render `Parent`, `Child`, and `NestedChild` upon every counter increment. However (as stated in "Micro State Management"), one can pass down static props through another
parent component to avoid redundant re-renders on elements that wouldn't change:

```jsx
function Parent(props) {
  // ...
  return (
    <div id='parent'>
      // ...
      {props.child}
    </div>
  );
}

function Grandparent() {
  return <Parent child={<Child />} />;
}

function App() {
  return <Grandparent />;
}
```

## What Happens When A Component Function Is Called

React does not force an update automatically, but evaluates whether or not an update is needed. As removing/updating/adding DOM is a performance-intensive task, React does not throw away the
current DOM and replace it with a new one, but instead employs the **virtual dom** to determine when and where an update is needed.

In general, JSX elements are stored as a JavaScript dictionary-like object that contains attributes such as element, children, etc. This will allow the comparison between the virtual DOM and
the real DOM to be much less performance-intensive. Whenever a component function is called, React compares the returned JSX code to the respective virtual DOM nodes; if differences are detected,
React will determine which changes are needed to update the DOM.

## State Batching

I.e., when multiple state updates are initiated from the same place in one's code.

```jsx
function App() {
  const [counter, setCounter] = useState(0);
  const [showCounter, setShowCounter] = useState(false);
  function incCounterHandler() {
    setCounter(prevCounter => prevCounter + 1);
    if (!showCounter) {
      setShowCounter(true);
    }
  }
  console.log();
  return (
    <>
      <p>Click to increment + show or hide the counter</p>
      <button onClick={incCounterHandler}>Increment</button>
      {showCounter && <p>Counter: {counter}</p>}
    </>
  );
}
```

`console.log()` will only be invoked once when `Increment` is clicked; this is state batching as React binds multiple state updates that are initiated from the same place in one's code
(e.g., from within the same event handler function) to prevent comparing the virtual DOM inefficiently (i.e., one at a time).

## Avoiding Unnecessary Child Evaluations

```jsx
function Error({ message }) {
  if (!message) {
    return null;
  }
  return <p className={classes.error}>{message}</p>;
}
function Form() {
  const [enteredEmail, setEnteredEmail] = useState('');
  const [errorMessage, setErrorMessage] = useState();
  function updateEmailHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function submitHandler(event) {
    event.preventDefault();
    if (!enteredEmail.endsWith('.com')) {
      setErrorMessage('Email must end with .com.');
    }
  }
  return (
    <form className={classes.form} onSubmit={submitHandler}>
      <div className={classes.control}>
        <label htmlFor='email'>Email</label>
        <input
          id='email'
          type='email'
          value={enteredEmail}
          onChange={updateEmailHandler}
        />
      </div>
      <Error message={errorMessage} />
      <button>Sign Up</button>
    </form>
  );
}
```

In this function, the `Error` component relies on the `message` prop, which depends on the `errorMessage` state; however, as the form also manages `enteredEmail`, changes to the
`enteredEmail` state will cause the `Error` component to be executed again, despite not being needed.

This is a sample fix:

```jsx
function Error({ message }) {
  console.log('<Error /> component function is executed.');
  if (!message) {
    return null;
  }
  return <p className={classes.error}>{message}</p>;
}
export default memo(Error);
```

When `memo` is invoked, React will check whether the component's props has change as compared to the last time the component function was called; if the prop values are equal,
the component function will not be executed again.

`memo` also takes an optional second argument to customize a logic to determine whether prop values have changed or not, e.g.:

```jsx
memo(SomeComponent, function (prevProps, nextProps) {
  return prevProps.user.firstName !== nextProps.user.firstName;
});
```

`memo` are typically only used in a few selected components, as avoiding unnecessary component re-evaluations with `memo` also requires some code to run; hence, `memo` is only
for relatively simple props (no deeply nested objects) but not too simple so that using `memo` will not yield any measurable benefit. If the component function must be executed
again, one will need extra time comparing props just to invoke the component function.

The usage on the `Error` component is apt because most state changes in the parent component won't affect `Error`, and the prop comparison will be simple given it's just
one prop; but still, `Error` is an extremely basic component with no complex logic, so either using/not using `memo` will be fine.

A good spot to use `memo` is a component that's close to the top of the component tree, or a deeply nested branch of components in the component tree; one would also implicitly
avoid unnecessary executions of all nested components beneath that one component. Another use case will be sorting a long list.

## Avoiding Costly Computations

Consider the following example:

```jsx
function sortItems(items) {
  console.log('Sorting');
  return items.sort(function (a, b) {
    if (a.id > b.id) {
      return 1;
    } else if (a.id < b.id) {
      return -1;
    }
    return 0;
  });
}
function List({ items, maxNumber }) {
  const sortedItems = sortItems(items);
  const listItems = sortedItems.slice(0, maxNumber);
  return (
    <ul>
      {listItems.map(item => (
        <li key={item.id}>
          {item.title} (ID: {item.id})
        </li>
      ))}
    </ul>
  );
}
```

Obviously, the list needs to be sorted whenever the items change, but not if `maxNumber` changes as it doesn't affect the list's order. However, with the code snippet above, `sortItems` will be
executed whenever either prop value changes. `memo` won't help because `List` component will execute whenever either variable changes, and `memo` does not control partial code execution inside
the component function.

One can use the `useMemo` hook in this case, which wraps a compute-intensive computation. It's similar to `useEffect`, but `useMemo` runs at the same time as the rest of the code in the component
function, whereas `useEffect` executes the wrapped logic after the compunent function execution finished (i.e., the dependency list).

`useMemo`, unlike `useEffect` (which controls side effects), controls the execution of performance-intensive tasks:

```jsx
import { useMemo } from 'react';
function List({ items, maxNumber }) {
  const sortedItems = useMemo(
    function () {
      console.log('Sorting');
      return items.sort(function (a, b) {
        // ...
      });
    },
    [items]
  );
  const listItems = sortedItems.slice(0, maxNumber);
  return (
    <ul>
      {listItems.map(item => (
        // ...
      ))}
    </ul>
  );
}
```

`useMemo` wraps an anonymous function which contains the sorting code, and the second argument is a dependency array for the function to be executed again. In this case, the sorting logic will only
execute when items change, not when anything else changes. `useMemo` can be an addon for `memo`, where the latter controls the overall component function execution. But also like `memo`, only wrap
very computation-intensive tasks inside `useMemo`, as calculating dependency changes also comes at a dependency cost.

## `useCallback`

`useCallback` can be used to avoid unnecessary code execution, in cases where a function is passed as a prop. For instance:

```jsx
function Error({ message, onClearError }) {
  console.log('<Error /> component function is executed.');
  if (!message) {
    return null;
  }
  return (
    <div className={classes.error}>
      <p>{message}</p>
      <button className={classes.errorBtn} onClick={onClearError}>
        X
      </button>
    </div>
  );
}
```

```jsx
function Form() {
  const [enteredEmail, setEnteredEmail] = useState('');
  const [errorMessage, setErrorMessage] = useState();
  function updateEmailHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function submitHandler(event) {
    event.preventDefault();
    if (!enteredEmail.endsWith('.com')) {
      setErrorMessage('Email must end with .com.');
    }
  }
  function clearErrorHandler() {
    setErrorMessage(null);
  }
  return (
    <form className={classes.form} onSubmit={submitHandler}>
      <div className={classes.control}>
        <label htmlFor='email'>Email</label>
        <input
          id='email'
          type='email'
          value={enteredEmail}
          onChange={updateEmailHandler}
        />
      </div>
      <Error message={errorMessage} onClearError={clearErrorHandler} />
      <button>Sign Up</button>
    </form>
  );
}
```

As functions are objects in JS, a new `Error` will be created every time `clearErrorHandler` is recreated, given that a new function will be a different, new instance. Therefore, `memo` cannot
prevent component function execution anymore. I.e., to `memo`, the old and new `clearErrorHandler` function objects are different from each other. Here's where `useCallback` comes in:

```jsx
const clearErrorHandler = useCallback(() => {
  setErrorMessage(null);
}, []);
```

By employing the `useCallback` hook, the re-creation of the function is prevented, and `memo` can interpret that the old and new `onClearError` prop values are equal and prevents unnecessary
function component executions. Recall from the previous chapter:

> `useCallback` does not execute the received function immediately but ensures that a function is only recreated if one of the specified dependencies
> has changed; a new function object will not be created (memoized, in a certain way).

`useCallback` can also be used with `useMemo`, where `useMemo` wraps a compute-intensive operation and `useCallback` prevents a dependent function from being unnecessarily recreated.

## Avoiding Unnecessary Code Download

Preventing clients from downloading large code bundles will also improve the website's loading speed. There are three main approaches:

1. Write short and concise code
2. Be thoughtful about including lots of third-party libraries (& don't use if unnecessary)
3. Consider code-splitting

## Code Spliting (Lazy Loading)

Lazy-loading can be used to load a component code conditionally, i.e., only when it's needed, through:

```jsx
const DateCalculator = lazy(() => import('./components/DateCalculator'));
```

and load through:

```jsx
<Suspense fallback={<p>Loading...</p>}>
  {showDateCalc && <DateCalculator />}
</Suspense>
```

`Suspense` is a React component that must be wrapped around React's `lazy` function; suspense must also provide a callback, which is a JSX value that will be rendered before the dynamic content is available.

## Strict Mode

Enables some extra checks that are performed under the hood by React. Most checks perform checks on unsafe or legacy mode, but also some that identify potential problems within one's code. For example, when
using strict mode, React will execute component functions twice and unmount/remount every component after it mounts for the first time. Notably, using strict mode may lead to confusing or annoying error messages,
e.g., when component effects are executing too often.
