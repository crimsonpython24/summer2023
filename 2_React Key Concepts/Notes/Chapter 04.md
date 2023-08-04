# Chapter 4

## Reacting to Events

```jsx
function EmailInput() {
  let errorMessage = '';
  function evaluateEmail(event) {
    const enteredEmail = event.target.value;
    if (enteredEmail.trim() === '' || !enteredEmail.includes('@')) {
      errorMessage = 'The entered email address is invalid.';
    } else {
      errorMessage = '';
    }
  }
  return (
    <div>
      <input placeholder='Your email' type='email' onBlur={evaluateEmail} />
      <p>{errorMessage}</p>
    </div>
  );
}
```

> `onBlur` is a function that triggers whenever the text input is not in focus

## Updating State Correctly

Props are about using external data within a component, but state is about managing and updating internal data. Modifying the code from above:

```jsx
function EmailInput() {
  const [errorMessage, setErrorMessage] = useState('');
  function evaluateEmail(event) {
    const enteredEmail = event.target.value;
    if (enteredEmail.trim() === '' || !enteredEmail.includes('@')) {
      setErrorMessage('The entered email address is invalid.');
    } else {
      setErrorMessage('');
    }
  }
  return (
    <div>
      <input placeholder='Your email' type='email' onBlur={evaluateEmail} />
      <p>{errorMessage}</p>
    </div>
  );
}
```

The core idea behind state management: state is data, and when the data is changed, React should be forced to re-render a component and update the UI
(which may cause extra re-renders, but see "Micro State Management" Chapter 3 for more details)

## Naming Conventions

```jsx
const [enteredEmail, setEnteredEmail] = useState('');
```

## Handling Complex State

### Using Multiple State Slices

```jsx
function LoginForm() {
  const [enteredEmail, setEnteredEmail] = useState('');
  const [enteredPassword, setEnteredPassword] = useState('');
  function emailEnteredHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function passwordEnteredHandler(event) {
    setEnteredPassword(event.target.value);
  }
  // Below, props are split across multiple lines for better readability
  // This is allowed when using JSX, just as it is allowed in standard
  HTML;
  return (
    <form>
      <input
        type='email'
        placeholder='Your email'
        onBlur={emailEnteredHandler}
      />
      <input
        type='password'
        placeholder='Your password'
        onBlur={passwordEnteredHandler}
      />
    </form>
  );
}
```

Advantage: they can be updated independently (but honestly, the entire component will still re-render given the change -- see "Micro State Management"
for more details) but independent updating can make large states more manageable

Disadvantage: many repeated `useState` calls can bloat a software really quickly.

### One Big State Object

```jsx
function LoginForm() {
  const [userData, setUserData] = useState({
    email: '',
    password: '',
  });
  function emailEnteredHandler(event) {
    setUserData({
      email: event.target.value,
      password: userData.password,
    });
  }
  function passwordEnteredHandler(event) {
    setUserData({
      email: userData.email,
      password: event.target.value,
    });
  }
  // same returned JSX code
}
```

One important note is that _all_ variales should be set, even for ones that don't change, as the new object will overwrite the old one (so that's where
the `...` deconstructor gets used to "migrate" the old properties and overwrite them with a second argument).

Big state components are okay _as long as_ they achieve the same purpose and do not try to complete too many things at once.

## Using State Based On a Previous State

```jsx
function Counter() {
  const [counter, setCounter] = useState(0);
  function incrementCounterHandler() {
    setCounter(prevCounter => prevCounter + 1);
  }
  return (
    <>
      <p>Counter Value: {counter}</p>
      <button onClick={incrementCounterHandler}>Increment</button>
    </>
  );
}
```

The arrow function here is used to "borrow" the previous value in the event that the `counter` state variable does not update immediately.

For the previous example, one can also rewrite the event handlers so that they will not use stale values:

```jsx
// old function
function emailEnteredHandler(event) {
  setUserData({
    email: event.target.value,
    password: userData.password,
  });
}

// new function
function emailEnteredHandler(event) {
  setUserData(prevData => ({
    email: event.target.value,
    password: prevData.password,
  }));
}
```

## Two-Way Binding

Two-way binding is when a value is both set and read from the same source; this mechanism is to prevent the UI and the state library from having
different values. As seen in `[1]` and `[2]`:

```jsx
function NewsletterField() {
  const [email, setEmail] = useState('');
  function changeEmailHandler(event) {
    setEmail(event.target.value);
  }
  return (
    <>
      <input
        type='email'
        placeholder='Your email address'
        value={email} // [1]
        onChange={changeEmailHandler} // [2]
      />
    </>
  );
}
```

## Deriving Values from State

COnsider the following piece of code:

```jsx
function CharCounter() {
  const [userInput, setUserInput] = useState('');
  function inputHandler(event) {
    setUserInput(event.target.value);
  }
  const numChars = userInput.length;
  return (
    <>
      <input type='text' onChange={inputHandler} />
      <p>Characters entered: {numChars}</p>
    </>
  );
}
```

This code, namely the `numChars` constant, will automatically update because calling a state-updating function such as `setUserInput` will force
React to re-evaluate the compoonent to which the state belongs to, simultaneously updating all the values inside `CharCounter`.

## Working with Forms and Form Submission

The `onSubmit` prop can be added to assign a function that will be executed once a form is submitted. For instance:

```jsx
function NewsletterSignup() {
  const [email, setEmail] = useState('');
  const [agreed, setAgreed] = useState(false);
  function updateEmailHandler(event) {
    // could add email validation here
    setEmail(event.target.value);
  }
  function updateAgreementHandler(event) {
    setAgreed(event.target.checked);
  }
  function signupHandler(event) {
    event.preventDefault(); // prevent browser default of sending a Http request;
    const userData = { userEmail: email, userAgrees: agreed };
    // can send the data to a backend server
  }
  return (
    <form onSubmit={signupHandler}>
      <div>
        <label htmlFor='email'>Your email</label>
        <input type='email' id='email' onChange={updateEmailHandler} />
      </div>
      <div>
        <input type='checkbox' id='agree' onChange={updateAgreementHandler} />
        <label htmlFor='agree'>Agree to terms and conditions</label>
      </div>
    </form>
  );
}
```

The `preventDefault` prevents the browser from generating an HTTP request. This function will stop a function from performing its original
action; specifically, `preventDefault()` on form submission will prevent the page from reloading.

## Lifting State Up

Consider this code segment:

```jsx
function SearchBar() {
  const [searchTerm, setSearchTerm] = useState('');
  function updateSearchTermHandler(event) {
    setSearchTerm(event.target.value);
  }
  return <input type='search' onChange={updateSearchTermHandler} />;
}
function Overview() {
  return <p>Currently searching for {searchTerm}</p>;
}
function App() {
  return (
    <>
      <SearchBar />
      <Overview />
    </>
  );
}
```

The `Overview` component should display the text within `SearchBar`; however, they are in two separate components. The most common solution is to
"lift" the state up, i.e., moving the shared data to the closest parent component, which will be `App`.

```jsx
function SearchBar(props) {
  return <input type='search' onChange={props.onUpdateSearch} />;
}
function Overview({ currentTerm }) {
  return <p>Currently searching for {currentTerm}</p>;
}
function App() {
  const [searchTerm, setSearchTerm] = useState('');
  function updateSearchTermHandler(event) {
    setSearchTerm(event.target.value);
  }
  return (
    <>
      <SearchBar onUpdateSearch={updateSearchTermHandler} />
      <Overview currentTerm={searchTerm} />
    </>
  );
}
```

Recall that, once `setSearchTerm` is invoked in `updateSearchTermHandler`, the component will refresh, and the `searchTerm` variable will be
updated, therefore refreshing the value prop passed into `Overview`.
