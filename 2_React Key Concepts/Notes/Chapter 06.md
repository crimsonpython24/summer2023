# Chapter 6

> Most content is deemed as preliminary knowledge and are therefore omitted

## Merging Inline Styles

```jsx
function ExplanationText({ children, isImportant }) {
  let defaultStyle = { color: 'black' };
  if (isImportant) {
    defaultStyle = { ...defaultStyle, backgroundColor: 'red' };
  }
  return <p style={defaultStyle}>{children}</p>;
}
```

Using the deconstructor (above) or using string literals (below):

```jsx
function Button({ children, config, className }) {
  return (
    <button {...config} className={`btn ${className}`}>
      {children}
    </button>
  );
}

<Button config={{ onClick: doSomething }} className='btn-alert'>
  Click me!
</Button>;
```
