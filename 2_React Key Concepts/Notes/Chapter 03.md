# Chapter 3

## Consuming Props

From this:

```jsx
<ul>
  <GoalItem />
  <GoalItem />
</ul>
```

Into this:

```jsx
<ul>
  <GoalItem id='g1' title='Finish the book!' />
  <GoalItem id='g2' title='Learn all about React!' />
</ul>;

function GoalItem(props) {
  return (
    <li>
      {props.title} (ID: {props.id})
    </li>
  );
}
```

The name doesn't have to be `props` but it's a convention.

## The `children` Prop

Sometimes, it can be more intuitive to pass things in between arrow brackets rather than passing them in props. For instance:

```jsx
<GoalItem id='g1'>Learn React</GoalItem>
```

In that case, one can obtain the information in between the arrow brackets with the `props.children` identifier:

```jsx
function GoalItem(props) {
  return (
    <li>
      {props.children} (ID: {props.id})
    </li>
  );
}
```

Notably, one don't need to explicitly pass `props` as an argument if related attributes are not used in the function.

## Dealing with Multiple Props

```jsx
<Product title="A book" price={29.99} id="p1" />
// or
const productData = {title: 'A book', price: 29.99, id: 'p1'}
<Product data={productData} />
```

The reverse can also be done (i.e., de-concatenate a dictionary into individual variable identifiers):

```jsx
const user = { name: 'Max', age: 29 };
const { name, age } = user;
console.log(name); // outputs 'Max'
```

## Spreading Props

As a side note, these two notations are equal due to destructuring, where `props` is destructured into individual variables:

```jsx
function Link(props) {
  return (
    <a target='_blank' rel='noopener noreferrer'>
      {props.children}
    </a>
  );
}

function Link({ children }) {
  return (
    <a target='_blank' rel='noopener noreferrer'>
      {children}
    </a>
  );
}
```

If there is an extra `href` prop in the function call, then one can pass the variable name directly into the deconstructured dictionary that is
used in replacement for props:

```jsx
<Link href='https://some-site.com'>Click here</Link>;

function Link({ children, href }) {
  return (
    <a href={href} target='_blank' rel='noopener noreferrer'>
      {children}
    </a>
  );
}
```

Or as a more universal solution, use the standard spread operator:

```jsx
function Link({ children, config }) {
  return (
    <a {...config} target='_blank' rel='noopener noreferrer'>
      {children}
    </a>
  );
}
```

## Prop Drilling
