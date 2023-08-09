# Chapter 8

## The Problem without `useEffect`

Consider this following code segment:

```jsx
async function fetchPosts() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  const blogPosts = await response.json();
  return blogPosts;
}
function BlogPosts() {
  const [loadedPosts, setLoadedPosts] = useState([]);
  fetchPosts().then(fetchedPosts => setLoadedPosts(fetchedPosts));
  return (
    <ul className={classes.posts}>
      {loadedPosts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
export default BlogPosts;
```

This code will infinitely send requests to the link above. As the response brings back new posts, the component is re-rendered via the `setState` hook; and
since the component receive new posts and is updated, the component will re-render, sending another request to fetch data from the API. I.e., the side
effect of fetching the data will result in an infinite loop and send a tremendous number of requests behind the scenes.

Side effects don't have to be `fetch` requests. It can be anything that does not directly relate to the function's main purpose.

As a side note, the `fetch` above is a side effect because it does not directly affect the UI's rendering. If the `fetch` statement is a necessary
part of the UI, then it will not be a side effect. This is illustrated in the following example:

```jsx
function BlogPosts() {
  const [loadedPosts, setLoadedPosts] = useState([]);
  function fetchPostsHandler() {}
  fetchPosts().then(fetchedPosts => setLoadedPosts(fetchedPosts)); // [1]
  return (
    <>
      <button onClick={fetchPostsHandler}>Fetch Posts</button>
      <ul className={classes.posts}>
        {loadedPosts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </>
  );
}
```

As the fetch in `[1]` is an integrated function with the `button` element, it is not a side effect.

## `useEffect`

```jsx
async function fetchPosts() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  const blogPosts = await response.json();
  return blogPosts;
}
function BlogPosts() {
  const [loadedPosts, setLoadedPosts] = useState([]);
  useEffect(function () {
    fetchPosts().then(fetchedPosts => setLoadedPosts(fetchedPosts));
  }, []);
  return (
    <ul className={classes.posts}>
      {loadedPosts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

`useEffect` should be used to extract side effects from the main component. The first argument is just a function, and the second argument specifies
the frequency to which the function will update. If the second argument is:

- an empty bracket `[]`, then there are no _dependencies_, i.e. the function will only be executed once, when the function is rendered the first time
- a list `[item1, item2]`, then there are dependencies, where the `useEffect` hook will be re-executed once the variables in the dependencies list change
- not provided, then React will execute teh effect function after every component re-evaluation

## Cleanup

Take an example where an alert sets a timer:

```jsx
function Alert() {
  const [alertDone, setAlertDone] = useState(false);
  useEffect(function () {
    console.log('Starting Alert Timer!');
    setTimeout(function () {
      console.log('Timer expired!');
      setAlertDone(true);
    }, 2000);
  }, []);
  return (
    <>
      {!alertDone && <p>Relax, you still got some time!</p>}
      {alertDone && <p>Time to get up!</p>}
    </>
  );
}
export default Alert;
```

```jsx
function App() {
  const [showAlert, setShowAlert] = useState(false);
  function showAlertHandler() {
    // state updating is done by passing a function to setShowAlert
    // because the new state depends on the previous state (it's the opposite)
    setShowAlert(isShowing => !isShowing);
  }
  return (
    <>
      <button onClick={showAlertHandler}>
        {showAlert ? 'Hide' : 'Show'} Alert
      </button>
      {showAlert && <Alert />}
    </>
  );
}
```

This function above will **not** clear the timer accurately. More specifically, in `{showAlert && <Alert/>}`, even if `showAlert` becomes `false`, the
timer will keep running because the `useEffect` component in `Alert` is not cleared.

In more detail, `setShowAlert` in the button will cause the `App` component to be re-evaluated, therefore causing `<Alert/>` to also be re-evaluated. It's
necessary to have the `useEffect` hook in `<Alert/>` be cleaned up every time, otherwise there will be many timers running simulatenously if the cleanup is
not implemented correctly. In other words, every time `<Alert/>` is called, the entire alert component should be re-evaluated and the `useEffect` hook
reset before the hook (aka the timer) is called again. Note that the cleanup function will be called each time the component **unmounts** or if some
dependency changes.

In that case, the effect function passed as a first argument can return an optional cleanup function; the said function will be executed _before_ React runs
the effect again. Consider the following revision:

```jsx
useEffect(function () {
  let timer;
  console.log('Starting Alert Timer!');
  timer = setTimeout(function () {
    console.log('Timer expired!');
    setAlertDone(true);
  }, 2000);
  return function () {
    clearTimeout(timer);
  };
}, []);
```

In the function returned by the effect function, the timer is removed, as the cleanup function will be executed automatically by React before the effect
function is called the next time. The cleanup function will be called by react whenever a component that contains an effect unmounts (i.e., removed from
the DOM).

## Dealing with Multiple Effects

It's a good approach to split effect functions by dependencies:

```jsx
function Demo() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  useEffect(
    function () {
      console.log(a);
    },
    [a]
  );
  useEffect(
    function () {
      console.log(b);
    },
    [b]
  );
}
```

But the best approach is to split effect functions by _logic_, e.g., separate an effect that deals with fetching data via an HTTP request from an effect
that sets a timer.

## Passing a Function as a Dependency

Yes, one can do that.

```jsx
function Alert() {
  const [alertMsg, setAlertMsg] = useState('Expired!');
  function changeAlertMsgHandler(event) {
    setAlertMsg(event.target.value);
  }
  function setAlert() {
    return setTimeout(function () {
      console.log(alertMsg);
    }, 2000);
  }
  useEffect(
    function () {
      const timer = setAlert();
      return function () {
        clearTimeout(timer); // only the final message entered will be printed
      };
    },
    [setAlert]
  );
  return <input type='text' onChange={changeAlertMsgHandler} />;
}
```

## Avoiding Unnecessary Effect Executions

Consider the following flawed code:

```jsx
function Alert() {
  const [enteredEmail, setEnteredEmail] = useState('');
  const [enteredPassword, setEnteredPassword] = useState('');
  function updateEmailHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function updatePasswordHandler(event) {
    setEnteredPassword(event.target.value);
  }
  function validateEmail() {
    if (!enteredEmail.includes('@')) {
      console.log('Invalid email!');
    }
  }
  useEffect(
    function () {
      validateEmail();
    },
    [validateEmail]
  );
  return (
    <form>
      <div>
        <label>Email</label>
        <input type='email' onChange={updateEmailHandler} />
      </div>{' '}
      <div>
        <label>Password</label>
        <input type='password' onChange={updatePasswordHandler} />
      </div>
      <button>Save</button>
    </form>
  );
}
```

The code above will create unnecessary effects: since `validateEmail` will change every time the component is re-rendered (memory values will be
re-assigned and the functions are re-declared, so technically it will become a different function), the effect function will be
executed because `validateEmail` changes. Either an email or a password change will cause this behavior, even if password is completely irrelevant to
the `validateEmail` field. There are two solutions:

- move `validateEmail` inside `useEffect` and make `enteredEmail` the only dependency
- avoid using `useEffect` altogether and perform email validation inside `updateEmailHandler`

or utilize `useCallback` to validate the email instead:

```jsx
const validateEmail = useCallback(
  function () {
    if (!enteredEmail.includes('@')) {
      console.log('Invalid email!');
    }
  },
  [enteredEmail]
);
```

`useCallback` does not execute the received function immediately but ensures that a function is only recreated if one of the specified dependencies
has changed; a new function object will not be created (memoized, in a certain way). In this method, the effect function will only execute when
`enteredEmail` changes as it's the callback's only dependency but not when `enteredPassword` changes.

Another way to avoid unnecessary executions is to not pass an entire object into `useEffect`'s second argument. Take this code for example:

```jsx
function Error(props) {
  const { message } = props; // destructure to extract required
  properties;
  useEffect(
    function () {
      console.log('An error occurred!');
      console.log(props.message);
    },
    // [props] // don't use the entire props object!
    [message]
  );
  return <p>{props.message}</p>;
}
```

Once again because component re-evaluations will produce brand-new JavaScript objects; even if the values of those objects do not change, React will
view the object as being brand new and create a new Effect.

## Effects and Async Code

Notably, when performing async tasks in Effect hooks, the function itself should not return a promise. There are two solutions:

One is to use promises without `async` and `await`

```jsx
useEffect(function () {
  fetchPosts.then(fetchedPosts => setLoadedPosts(fetchedPosts));
}, []);
```

Or create a separate wrapper function inside the effect function:

```jsx
useEffect(function () {
  async function loadData() {
    const fetchedPosts = await fetchPosts();
    setLoadedPosts(fetchedPosts);
  }
  loadData();
}, []);
```

## Rules of Hooks

1. Only call hooks at the top level of component functions; do not call them inside `if` statments, loops, or nested functions
2. Only call hooks inside of React components of custom Hooks
