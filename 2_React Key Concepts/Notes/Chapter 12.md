# Chapter 12

> Most details are skipped as they're considered preliminary knowledge

## Route vs. "Normal" Components

`src/routes` is the most common place to store Route components, as opposed to `src/components` for regular components.

## Undefined Routes

```jsx
<Route path='*' element={<NotFound />} />
```

## Lazy Loading

```jsx
const OrdersRoot = lazy(() => import('./routes/OrdersRoot'));
const Orders = lazy(() => import('./routes/Orders'));
const OrderDetail = lazy(() => import('./routes/OrderDetail'));

<Suspense fallback={<p>Loading...</p>}>
  <Routes>
    <Route path='/' element={<Dashboard />} />
    <Route path='/orders' element={<OrdersRoot />}>
      <Route element={<Orders />} index />
      <Route path=':id' element={<OrderDetail />} />
    </Route>
  </Routes>
</Suspense>;
```
