# Chapter 7

Instead of manipulating the DOM or reading values from DOM elements manually, one can use React to execute such changes; however, if one still wants
to reach out to specific DOM elements (e.g., reading an user input), one can use **portals** and **refs**.

## Basics

Consider an email form that mixes vanilla and React.JS:

```jsx
function EmailForm() {
  const [enteredEmail, setEnteredEmail] = useState('');
  function updateEmailHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function submitFormHandler(event) {
    event.preventDefault();
    // could send enteredEmail to a backend server
  }
  return (
    <form className={classes.form} onSubmit={submitFormHandler}>
      <label htmlFor='email'>Your email</label>
      <input type='email' id='email' onChange={updateEmailHandler} />
      <button>Save</button>
    </form>
  );
}
```

**Refs** can read values inside DOM elements

```jsx
function EmailForm() {
  const emailRef = useRef();
  function submitFormHandler(event) {
    event.preventDefault();
    const enteredEmail = emailRef.current.value; // [1]
    // could send enteredEmail to a backend server
  }
  return (
    <form className={classes.form} onSubmit={submitFormHandler}>
      <label htmlFor='email'>Your email</label>
      <input ref={emailRef} type='email' id='email' />
      <button>Save</button>
    </form>
  );
}
```

For `[1]`, as React stores the referenced object's value in a nested object, one must access the value through the `current` property.

## Refs vs. State

Refs should be used when one only needs to read the value of a DOM element. A counter example is when the email form should also be cleared,
where `useState` will be the preferred approach:

```jsx
function EmailForm() {
  const [enteredEmail, setEnteredEmail] = useState('');
  function updateEmailHandler(event) {
    setEnteredEmail(event.target.value);
  }
  function submitFormHandler(event) {
    event.preventDefault();
    // could send enteredEmail to a backend server
    // reset by setting the state + using the value prop below
    setEnteredEmail('');
  }
  return (
    <form className={classes.form} onSubmit={submitFormHandler}>
      <label htmlFor='email'>Your email</label>
      <input
        type='email'
        id='email'
        onChange={updateEmailHandler}
        value={enteredEmail}
      />
      <button>Save</button>
    </form>
  );
}
```

In `[1]`, using `emailRef.current.value = ''` is not a recommended method, as ref objects should be read-only and not manipulated.

## Extra Functionalities

One can store data that "survive" component re-evaluations with `useRef`, as refs are not reset or cleared when the surrounding component
function is executed again and the entire component is re-rendered, whereas vanilla JS variables will not retain their value.

```jsx
function Counters() {
  const [counter1, setCounter1] = useState(0);
  const counterRef = useRef(0);
  let counter2 = 0;
  function changeCountersHandler() {
    setCounter1(1);
    counter2 = 1;
    counterRef.current = 1;
  }
  return (
    <>
      <button onClick={changeCountersHandler}>Change Counters</button>
      <ul>
        <li>Counter 1: {counter1}</li>
        <li>Counter 2: {counter2}</li>
        <li>Counter 3: {counterRef.current}</li>
      </ul>
    </>
  );
}
```

However, if `counter1` is changed to be using ref:

```jsx
function changeCountersHandler() {
  counterRef1.current = 1;
  counter2 = 1;
  counterRef2.current = 1;
}
```

Then this component will _not_ re-render because `set` hooks are the ones that will trigger a re-render and therefore cause
the component to be re-evaluated. So now, even if the Ref value is updated, the UI will not update and the updated value will
not be displayed. I.e., changes to ref values do **not** cause component re-render.

```jsx
const passwordRetries = useRef(0);
passwordRetries.current = 1; // changed from 0 to 1
console.log(passwordRetries.current); // prints 1, even if the component changed
```

**Conclusion** if the data

- should survive component re-evaluations
- should not be managed as state (because changes to that data should _not_ cause the component to be re-evaluated)

then one can use a ref. Note that this feature isn't used frequently, but it may be helpful from time to time

## Ref Forwarding

Suppose there is a form that has a `Preferences` element consisting of two checkboxes and an email input. The two checkboxes
should be cleared every time the email address is submitted.

```jsx
const Preferences = forwardRef((props, ref) => {
  // component code ...
});
export default Preferences;
```

If the checkboxes are in the same element as the form itself, then ref forwarding is unnecessary. However, as the checkboxes
live in the `Preferences` element and not directly under the form itself, ref forwarding is needed to pass refs between the
two levels. For the form itself (spoiler, this is a bad example, and the reason is explained in the following section):

```jsx
function Form() {
  const preferencesRef = useRef({});
  const [email, setEmail] = useState('');
  function submitHandler(event) {
    event.preventDefault();
    preferencesRef.current.reset();
    setEmail('');
  }
  function changeEmailHandler(event) {
    setEmail(event.target.value);
  }
  return (
    <form onSubmit={submitHandler}>
      <div>
        <label htmlFor='email'>Your email</label>
        <input
          type='email'
          id='email'
          onChange={changeEmailHandler}
          value={email}
        />
      </div>
      <Preferences ref={preferencesRef} />
      <button>Submit</button>
    </form>
  );
}
```

`useRef` is used to create a `ref` object that will be passed to the `Preferences` component.

```jsx
const Preferences = forwardRef((props, ref) => {
  const [wantsNewProdInfo, setWantsNewProdInfo] = useState(false);
  const [wantsProdUpdateInfo, setWantsProdUpdateInfo] = useState(false);
  function changeNewProdPrefHandler() {
    setWantsNewProdInfo(prevPref => !prevPref);
  }
  function changeUpdateProdPrefHandler() {
    setWantsProdUpdateInfo(prevPref => !prevPref);
  }
  function reset() {
    setWantsNewProdInfo(false);
    setWantsProdUpdateInfo(false);
  }
  ref.current.reset = reset;
  ref.current.selectedPreferences = {
    newProductInfo: wantsNewProdInfo,
    productUpdateInfo: wantsProdUpdateInfo,
  };
  return (
    <div className={classes.preferences}>
      <label>
        <input
          type='checkbox'
          id='pref-new'
          checked={wantsNewProdInfo}
          onChange={changeNewProdPrefHandler}
        />
        <span>New Products</span>
      </label>
      <label>
        <input
          type='checkbox'
          id='pref-updates'
          checked={wantsProdUpdateInfo}
          onChange={changeUpdateProdPrefHandler}
        />
        <span>Product Updates</span>
      </label>
    </div>
  );
});
```

## Controlled vs. Uncontrolled Components

Using forward refs often lead to more imperative code in the end (step-by-step instructions), which leads to
**uncontrolled components** because React does not directly control the UI state; instead, values are read from
other components or the DOM.

Rewrite:

```jsx
function Preferences({ newProdInfo, prodUpdateInfo, onUpdateInfo }) {
  return (
    <div className={classes.preferences}>
      <label>
        <input
          type='checkbox'
          id='pref-new'
          checked={newProdInfo}
          onChange={onUpdateInfo.bind(null, 'pref-new')}
        />
        <span>New Products</span>
      </label>
      <label>
        <input
          type='checkbox'
          id='pref-updates'
          checked={prodUpdateInfo}
          onChange={onUpdateInfo.bind(null, 'pref-updates')}
        />
        <span>Product Updates</span>
      </label>
    </div>
  );
}
```

```jsx
function Form() {
  const [wantsNewProdInfo, setWantsNewProdInfo] = useState(false);
  const [wantsProdUpdateInfo, setWantsProdUpdateInfo] = useState(false);
  function updateProdInfoHandler(selection) {
    // using one shared update handler function is optional
    // you could also use two separate functions (passed to Preferences) as props
    if (selection === 'pref-new') {
      setWantsNewProdInfo(prevPref => !prevPref);
    } else if (selection === 'pref-update') {
      setWantsProdUpdateInfo(prevPref => !prevPref);
    }
  }
  function reset() {
    setWantsNewProdInfo(false);
    setWantsProdUpdateInfo(false);
  }
  function submitHandler(event) {
    event.preventDefault();
    // state values can be used here
    reset();
  }
  return (
    <form className={classes.form} onSubmit={submitHandler}>
      <div className={classes.formControl}>
        <label htmlFor='email'>Your email</label>
        <input type='email' id='email' />
      </div>
      <Preferences
        newProdInfo={wantsNewProdInfo}
        prodUpdateInfo={wantsProdUpdateInfo}
        onUpdateInfo={updateProdInfoHandler}
      />
      <button>Submit</button>
    </form>
  );
}
```

## Portals

A portal instructs react to insert a DOM element in a different place than where it would be normally be inserted. For instance:

```jsx
<body>
  <div id='root'></div>
  <div id='dialogs'></div>
</body>;

function ErrorDialog({ onClose }) {
  return createPortal(
    <>
      <div className={classes.backdrop}></div>
      <dialog className={classes.dialog} open>
        <p>
          This form contains invalid values. Please fix those errors before
          submitting the form again!
        </p>
        <button onClick={onClose}>Okay</button>
      </dialog>
    </>,
    document.getElementById('dialogs')
  );
}
export default ErrorDialog;
```

This element above will be inserted into the `dialogs` div ID.
