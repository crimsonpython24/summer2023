# Chapter 5

> Most content is deemed as preliminary knowledge and are therefore omitted

## Ternary Expressions

```jsx
<div>{showTerms ? <p>Our terms of use ...</p> : null}</div>
```

## Conditional Element Tags

```jsx
function Button({ isButton, config, children }) {
  if (isButton) {
    return <button {...config}>{children}</button>;
  }
  return <a {...config}>{children}</a>;
}
```

## Updating Lists

```jsx
function Todos() {
  const [todos, setTodos] = useState(['Learn React', 'Recommend this book']);
  function addTodoHandler() {
    setTodos(curTodos => [...curTodos, 'A new todo']);
  }
  return (
    <div>
      <button onClick={addTodoHandler}>Add Todo</button>
      <ul>
        {todos.map(todo => (
          <li key={todo}>{todo}</li> // most likely will have a more formal ID from the database
        ))}
      </ul>
    </div>
  );
}
```

Once again, this entire component will re-render once `setTodos` is invoked, but this is unnecessary as list items will not change by themselves. In other words,
when the new list item is created, the other list items should _not_ be re-rendered. This unintended re-render can be prevented by adding the `key` prop to each
list item. Keys should be unique (the code snippet above is only for demonstration) but will only serve this particular purpose in list items.
