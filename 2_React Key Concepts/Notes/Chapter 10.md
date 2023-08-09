# Chapter 10

## Lifting State Up

Usually used for shallow component trees sharing a common parent; in that case, move the shared state into the parent, and pass the state into each child component as props.
This method is not recommended for deep components since prop drilling may make the code less readable and also induce unnecessary re-renders on the other side of the
component tree. In addition, prop drilling also:

- prevents a component from being reusable (since props must be passed every time)
- has a lot of overhead code (i.e., code that accepts and forwards props)
- complicates refactoring

In short, prop drilling is only suitable for components that aren't reusable, and exist only in a certain, small section of the component tree.

## Using Context

The first step is to create a context object:

```jsx
createContext('Hello Context'); // a context with an initial string value
const BookmarkContext = createContext({
  bookmarkedArticles: [],
});
```

This code is usually placed in a separate context code file in a folder called `store`, but this organization is not strictly required.

In order to use context values in other components, you must first provide the value using the value returned by `createContext` through the `Provider` property. Note that
the `Provider` property expects a value prop that informs the Context of the Provider's value. For instance:

```jsx
function News() {
  const BookmarkContext = createContext({
    bookmarkedArticles: [],
    bookmarkArticle: () => {},
    unbookmarkArticle: () => {},
  });
  return (
    <BookmarkContext.Provider value={bookmarkCtxValue}>
      <Articles />
      <InfoSidebar />
    </BookmarkContext.Provider>
  );
}
```

One can also combine Context with `useState` to make the content dynamic:

```jsx
function News() {
  const [savedArticles, setSavedArticles] = useState([]);
  function addArticle(article) {
    setSavedArticles(prevSavedArticles => [...prevSavedArticles, article]);
  }
  function removeArticle(articleId) {
    setSavedArticles(prevSavedArticles =>
      prevSavedArticles.filter(article => article.id !== articleId)
    );
  }
  const bookmarkCtxValue = {
    bookmarkedArticles: savedArticles,
    bookmarkArticle: addArticle,
    unbookmarkArticle: removeArticle,
  };
  return (
    <BookmarkContext.Provider value={bookmarkCtxValue}>
      <Articles />
      <InfoSidebar />
    </BookmarkContext.Provider>
  );
}
```

In the code example above, whenever `savedArticles` change, the context value will also change. Now, any components nested in `Articles` or `InfoSidebar` components (or
their children) can access the dynamic context value without prop drilling.

To make the context value accessible by components inside the context's `Provider` component, React provides (hah) the `useContext` hook. The hook requires one argument,
being the context object created via `createContext`. The context values can be used like:

```jsx
function BookmarkSummary() {
  const bookmarkCtx = useContext(BookmarkContext);
  const numberOfArticles = bookmarkCtx.bookmarkedArticles.length;
  return (
    <>
      <p className={classes.summary}>{numberOfArticles} articles bookmarked</p>
      <ul className={classes.list}>
        {bookmarkCtx.bookmarkedArticles.map(article => (
          <li key={article.id}>{article.title}</li>
        ))}
      </ul>
    </>
  );
}
```

In this code, `useContext` received the `BookmarkContext` value (which is `bookmarkCtxValue`); the returned object is stored in the `bookmarkCtx` constant. Whenever the
context value changes (if `setSavedArticles` changes), then React will also re-execute `BookmarkSummary`, and `bookmarkCtx` will hold the latest state value.

## Context vs. Lifting State Up

1. Lift the state up if you only need to share state across one or two levels of component nesting
2. Use Context if you have a large/deep component tree
3. Use Context if you have a flat component tree but wants to be able to reuse the component

## Outsourcing Context Logic

Using the same example as above, one can move all the functions related to context management into a separate, dedicated component that will make the code easier to maintain:

```jsx
export function BookmarkContextProvider({ children }) {
  const [savedArticles, setSavedArticles] = useState([]);
  function addArticle(article) {
    setSavedArticles(prevSavedArticles => [...prevSavedArticles, article]);
  }
  function removeArticle(articleId) {
    setSavedArticles(prevSavedArticles =>
      prevSavedArticles.filter(article => article.id !== articleId)
    );
  }
  const bookmarkCtxValue = {
    bookmarkedArticles: savedArticles,
    bookmarkArticle: addArticle,
    unbookmarkArticle: removeArticle,
  };
  return (
    <BookmarkContext.Provider value={bookmarkCtxValue}>
      {children}
    </BookmarkContext.Provider>
  );
}
```

The component above encapsulates all the logic related to managing a list of bookmarked articles and creates the same context value. It can be used like this:

```jsx
function News() {
  return (
    <BookmarkContextProvider>
      <Articles />
      <InfoSidebar />
    </BookmarkContextProvider>
  );
}
```

## Combining Multiple Contexts

Especially in larger React apps, it's quite possible that one will work with context values that are unrelated to each other. React fully supports this: one can
provide multiple contexts in the same component or in different components. In that case, one will call `useContext` multiple times with different context values.

## Limitations of `useState`

Remember that one should not reference the current state for setting a new state value, but instead use the function form of the state-updating function. More specifically:

```jsx
setIsLoading(fetchedPosts ? false : true);
```

From the above into this:

```jsx
setHttpState(prevState => ({
  fetchedPosts: prevState.fetchedPosts,
  isLoading: prevState.fetchedPosts ? false : true,
  error: null,
}));
```

In this solution, one can move from multiple, individual state slices into a single, combined state value to ensure that all state values are grouped together and can
therefore be accessed safely when using the functional state-updating form.

## `useReducer`

`useReducer` is a hook that specializes in managing complex state object. It has to parameters: the current state value, and an action that was dispatched. A reducer function
also returns the new state. It's therefore called a reducer because it reduces the old state and an action to a new state.

> Recall that one shouldn't use `useState` to derive a new state based on _another state variable_, unless the two states are already grouped together in an object form.

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
```

Like all react hooks, it must be called inside a component function or other hooks.

`httpReducer` is a reducer function: it takes two arguments and returns different state objects for different action types. This reducer function takes care of all possible
state updates. The shape and structure of the action value is entirely up to the user, but it's often set to an object that contains a `type` property, used to perform different
actions for different types of actions (i.e., an identifier). One can also add extra data (e.g., `action.payload`) for the reducer to digest, for example:

```jsx
dispatch({ type: 'FETCH_SUCCESS', payload: posts });
```

One can use the above function as follows:

```jsx
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

The hook also returns a value with two elements, with the first element (`httpState` in this case) being the state value returned by the reducer function, and it's updated whenever
the reducer function executes again. The second element is a function that can be called to trigger a state update, conventionally named `dispatch`.
