# Chapter 11

In React apps, custom Hooks are regular JS functions that:

- start with `use` (e.g., `useState`, `useEffect`)
- does not only return JSX code

Custom hooks are special because one can call built-in and custom React hooks inside their function bodies and not only inside component functions. I.e., there will be errors
if one tries to call a hook in:

- a regular JavaScript function
- loops, conditions, or nested functions
- after a conditional return

Namely, a React function

1. follows React component naming convention, i.e. they must be Capitalized
2. takes a single `props` object argument
3. returns valid JSX

Like JSX, hooks are a way to extract logic from a component and make the logic reusable in other components.

## Simple Custom Hook

```jsx
function useCounter() {
  const [counter, setCounter] = useState(0);
  function increment() {
    setCounter(oldCounter => oldCounter + 1);
  }
  function decrement() {
    setCounter(oldCounter => oldCounter - 1);
  }
  return { counter, increment, decrement };
}
```

As the name starts with `use`, React will treat this function as a custom hook, and you won't get any error messages when using other Hooks in it. The hook can then be used as follows:

```jsx
function Demo1() {
  const { counter, increment, decrement } = useCounter();
  return (
    <>
      <p>{counter}</p>
      <button onClick={increment}>Inc</button>
      <button onClick={decrement}>Dec</button>
    </>
  );
}
```

Note that in custom hooks, _logic_ is shared instead of _state_. One should use Context to share state. In Hooks, every component has its own instance, where any state or side effects
are executed on a per-component basis.

## Custom Hooks with Parameters

```jsx
function useCounter(initialValue, incVal, decVal) {
  const [counter, setCounter] = useState(initialValue);
  function increment() {
    setCounter(oldCounter => oldCounter + incVal);
  }
  function decrement() {
    setCounter(oldCounter => oldCounter - decVal);
  }
  return { counter, increment, decrement };
}
```

```jsx
function Demo1() {
  const { counter, increment, decrement } = useCounter(1, 2, 1);
  return (
    <>
      <p>{counter}</p>
      <button onClick={increment}>Inc</button>
      <button onClick={decrement}>Dec</button>
    </>
  );
}
```

## Return Values

Custom hooks _may_ return values, but they don't have to. Some built-in hooks, `useState` and `useReducer` for example, returns arrays with a fixed number of elements, whereas `useRef`
returns an object (that always has a `current` property) and `useEffect` returns nothing.

A rewrite:

```jsx
function useCounter(initialValue, incVal, decVal) {
  const [counter, setCounter] = useState(initialValue);
  function increment() {
    setCounter(oldCounter => oldCounter + incVal);
  }
  function decrement() {
    setCounter(oldCounter => oldCounter - decVal);
  }
  return [counter, increment, decrement];
}
```

```jsx
function Demo1() {
  const [counter, increment, decrement] = useCounter(1, 2, 1);
  return (
    <>
      <p>{counter}</p>
      <button onClick={increment}>Inc</button>
      <button onClick={decrement}>Dec</button>
    </>
  );
}
```

## A More Complex Example

Using the logic from Chapter 10:

```jsx
const initialHttpState = {
  data: null,
  isLoading: false,
  error: null,
};
function httpReducer(state, action) {
  if (action.type === 'FETCH_START') {
    return {
      ...state, // copying the existing state
      isLoading: state.data ? false : true,
      error: null,
    };
  }
  if (action.type === 'FETCH_ERROR') {
    return {
      data: null,
      isLoading: false,
      error: action.payload,
    };
  }
  if (action.type === 'FETCH_SUCCESS') {
    return {
      data: action.payload,
      isLoading: false,
      error: null,
    };
  }
  return initialHttpState; // default value for unknown actions
}
function App() {
  const [httpState, dispatch] = useReducer(httpReducer, initialHttpState);
  // Using useCallback() to prevent an infinite loop in useEffect() below
  const fetchPosts = useCallback(async function fetchPosts() {
    dispatch({ type: 'FETCH_START' });
    try {
      const response = await fetch(
        'https://jsonplaceholder.typicode.com/posts'
      );
      if (!response.ok) {
        throw new Error('Failed to fetch posts.');
      }
      const posts = await response.json();
      dispatch({ type: 'FETCH_SUCCESS', payload: posts });
    } catch (error) {
      dispatch({ type: 'FETCH_ERROR', payload: error.message });
    }
  }, []);
  useEffect(
    function () {
      fetchPosts();
    },
    [fetchPosts]
  );
  return (
    <>
      <header>
        <h1>Complex State Blog</h1>
        <button onClick={fetchPosts}>Load Posts</button>
      </header>
      {httpState.isLoading && <p>Loading...</p>}
      {httpState.error && <p>{httpState.error}</p>}
      {httpState.data && <BlogPosts posts={httpState.data} />}
    </>
  );
}
```

The `httpReducer` and `useEffect` call can be outsourced into a custom hook.

```jsx
// httpReducer function and initialHttpState are omitted as they're they're the same
function useFetch(url) {
  const [httpState, dispatch] = useReducer(httpReducer, initialHttpState);
  const fetchPosts = useCallback(
    async function fetchPosts() {
      dispatch({ type: 'FETCH_START' });
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error('Failed to fetch posts.');
        }
        const posts = await response.json();
        dispatch({ type: 'FETCH_SUCCESS', payload: posts });
      } catch (error) {
        dispatch({ type: 'FETCH_ERROR', payload: error.message });
      }
    },
    [url]
  );
  useEffect(
    function () {
      fetchPosts();
    },
    [fetchPosts]
  );
  return httpState;
}
```

Note that, since `httpReducer` is a pure function, it can be safely left outside of the hook to prevent unnecessary creations of the function instance. If the function is impure, then
move the function into the hook so that it will function as intended.

Now `App` will look like this:

```jsx
function App() {
  const { data, isLoading, error } = useFetch(
    'https://jsonplaceholder.typicode.com/posts'
  );
  return (
    <>
      <header>
        <h1>Complex State Blog</h1>
      </header>
      {isLoading && <p>Loading...</p>}
      {error && <p>{error}</p>}
      {data && <BlogPosts posts={data} />}
    </>
  );
}
export default App;
```
