# Chapter 13

Data fetching and routing are tightly coupled: whenever a user visits a new page, most likely some data will need to be fetched. Some features also require data submission (e.g., forms), or interact
with browser APIs such as `localStorage`.

Most of the time, data is fetched when a route becomes active (i.e., a component is rendered for the first time). After submitting data, the user might need to be redirected to another page as well.

## Sending HTTP Requests

Sending HTTP requests _without_ React Router:

```jsx
import { useState, useEffect } from 'react';
function Posts() {
  const [loadedPosts, setLoadedPosts] = useState();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState();
  useEffect(() => {
    async function fetchPosts() {
      setIsLoading(true);
      try {
        const response = await fetch(
          'https://jsonplaceholder.typicode.com/posts'
        );
        if (!response.ok) {
          throw new Error('Fetching posts failed.');
        }
        const resData = await response.json();
        setLoadedPosts(resData);
      } catch (error) {
        setError(error.message);
      }
      setIsLoading(false);
    }
    fetchPosts();
  }, []);
  let content = <p>No posts found.</p>;
  if (isLoading) {
    // [1]
    content = <p>Loading...</p>;
  }
  if (error) {
    // [2]
    content = <p>{error}</p>;
  }
  if (loadedPosts) {
    // [1]
    content = (
      <ul className='posts'>
        {loadedPosts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    );
  }
  return (
    <main>
      <h1>Your Posts</h1>
      {content}
    </main>
  );
}
```

The list items are only displayed if `isLoading` is false (`[1]`) or if there is no `error` (`[2]`). This can also be done through React Router (spoiler: wouldn't work
yet), pseudocode as seen below.

```jsx
import { useLoaderData } from 'react-router-dom';
function Posts() {
  const loadedPosts = useLoaderData();
  return (
    <main>
      <h1>Your Posts</h1>
      <ul className='posts'>
        {loadedPosts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  );
}
export default Posts;
export async function loader() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  if (!response.ok) {
    throw new Error('Could not fetch posts');
  }
  return response;
}
```

Then, on the route definition, set an extra `loader` prop that will be executed by Router whenever this route is activated and before the component function executes:

```jsx
<Route path="/posts" element={<Posts />} loader={() => {...}} />

// or import the component and its loader function
<Route path="/posts" element={<Posts />} loader={postsLoader} />
```

The `loader` function can pretty much do anything, from sending an HTTP request to reaching out to browser storage. One should return the data that will be exposed to the component function. Notably,
since `loader` executes on the client side, do not try to run code that belongs to the server side, which may introduce bugs and security risks (e.g., exposing database credentials to the client).

Since the component that belongs to a `loader` needs the data returned by the `loader`, React Router offers the `useLoaderData()` hook. For example, if `loader` returns a `Promise`, `useLoaderData`
will provide the resolved data until `Promise` resolves; if `loader` returns an HTTP response object, `useLoaderData` will automatically extrac the response body and give access to the response's data.

Modifying the example above:

```jsx
import { useLoaderData } from 'react-router-dom';
function PostsList() {
  const loadedPosts = useLoaderData();
  return (
    <main>
      <h1>Your Posts</h1>
      <ul className='posts'>
        {loadedPosts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  );
}

// different file
function Posts() {
  return (
    <main>
      <h1>Your Posts</h1>
      <PostsList />
    </main>
  );
}
export async function loader() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  if (!response.ok) {
    throw new Error('Could not fetch posts');
  }
  return response;
}

// parent file
import Posts, { loader as postsLoader } from './pages/Posts';
const router = createBrowserRouter(
  createRoutesFromElements(
    <>
      <Route path='/' element={<Welcome />} />
      <Route path='/posts' element={<Posts />} loader={postsLoader} />
    </>
  )
);
function App() {
  return <RouterProvider router={router} />;
}
const router2 = createBrowserRouter([
  // alternative definition
  { path: '/', element: <Welcome /> },
  { path: '/posts', element: <Posts />, loader: postsLoader },
]);
```

> One can't use `useLoaderData` in components where loaders aren't defined.

## Loading Data for Dynamic Routes

```jsx
<Route path='/posts/:id' element={<PostDetails />} />;
const router = createBrowserRouter([
  // ... other routes
  { path: '/posts/:id', element: <PostDetails /> },
]);

// different file
function PostDetails() {
  const post = useLoaderData();
  return (
    <main id='post-details'>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </main>
  );
}
export default PostDetails;
export async function loader({ params }) {
  const response = await fetch(
    'https://jsonplaceholder.typicode.com/posts/' + params.id
  );
  if (!response.ok) {
    throw new Error('Could not fetch post for id ' + params.id);
  }
  return response;
}
```

## Loaders, Requests, and Client-Side Code

React Router creates a request object through the browser's built-in `Request` interface to be used as a value for the `request` property on the data object passed to the `loader` function. For example:

```jsx
export async function loader({ request }) {
  // e.g. for localhost:3000/posts?sort=desc
  const sortDirection = new URL(request.url).searchParams.get('sort');
  const response = await fetch(
    'https://example.com/posts?sorting=' + sortDirection
  );
  return response;
}
```

The following code can get access to search parameters in the current active page's URL.

## Reusing Data Across Routes

```jsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      { index: true, element: <Welcome /> },
      {
        path: '/posts',
        id: 'posts', // the id value is up to you
        element: <PostsLayout />,
        children: [
          { index: true, element: <Posts /> },
          { path: ':id', element: <PostDetails />, loader: postDetailLoader },
        ],
      },
    ],
  },
]);

// different file
function PostDetails() {
  const params = useParams();
  const posts = useRouteLoaderData('posts'); // provided an unique ID, data can be reused
  const post = posts.find(post => post.id.toString() === params.id);
  return (
    <main id='post-details'>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </main>
  );
}
```

## Handling Errors

```jsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    errorElement: <Error />, // used for any errors not handled by nested routes
    children: [
      { index: true, element: <Welcome /> },
      {
        path: '/posts',
        id: 'posts',
        element: <PostsLayout />,
        // used if /posts or /posts/:id throws an error errorElement: <PostsError />,
        loader: postsLoader,
        children: [
          { index: true, element: <Posts /> },
          { path: ':id', element: <PostDetails /> },
        ],
      },
    ],
  },
]);

// different file
function Error() {
  const error = useRouteError();
  return (
    <>
      <h1>Oh no!</h1>
      <p>An error occurred</p>
      <p>{error.message}</p>
    </>
  );
}
```

## Submitting Forms

Before React Router:

```jsx
function NewPost() {
  const titleInput = useRef();
  const textInput = useRef();
  const navigate = useNavigate();
  async function submitHandler(event) {
    event.preventDefault(); // prevent the browser from sending a HTTP request
    const enteredTitle = titleInput.current.value;
    const enteredText = textInput.current.value;
    const postData = {
      title: enteredTitle,
      text: enteredText,
    };
    await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      body: JSON.stringify(postData),
      headers: { 'Content-Type': 'application/json' },
    });
    navigate('/posts');
  }
  return (
    <form onSubmit={submitHandler}>
      <p>
        <label htmlFor='title'>Title</label>
        <input type='text' id='title' ref={titleInput} />
      </p>{' '}
      <p>
        <label htmlFor='text'>Text</label>
        <textarea id='text' rows={3} ref={textInput} />
      </p>
      <button>Save Post</button>
    </form>
  );
}
```

With React Router:

```jsx
import { Form, redirect } from 'react-router-dom';
function NewPost() {
  return (
    <Form method='post' id='post-form'>
      {' '}
      <p>
        <label htmlFor='title'>Title</label>
        <input type='text' id='title' name='title' />{' '}
      </p>
      <p>
        <label htmlFor='text'>Text</label>{' '}
        <textarea id='text' rows={3} name='text' />
      </p>
      <button>Save Post</button>
    </Form>
  );
}
export default NewPost;
export async function action({ request }) {
  const formData = await request.formData();
  const postData = Object.fromEntries(formData);
  await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify(postData),
    headers: { 'Content-Type': 'application/json' },
  });
}
return redirect('/posts');
```

Just like `loader`, `action` is a special function that can be added to route definitions:

```jsx
import NewPost, { action as newPostAction } from './components/NewPost';
{ path: 'new', element: <NewPost />, action: newPostAction },
```

The extracted data can be used for purposes such as sending an HTTP request to some backend API:

```jsx
export async function action({ request }) {
  const formData = await request.formData();
  const postData = Object.fromEntries(formData);
  await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify(postData),
    headers: { 'Content-Type': 'application/json' },
  });
  return redirect('/posts');
}
```

## Handling Errors

```jsx
export async function action({ request }) {
  const formData = await request.formData();
  const postData = Object.fromEntries(formData);
  let validationErrors = [];
  if (postData.title.trim().length === 0) {
    validationErrors.push('Invalid post title provided.');
  }
  if (postData.text.trim().length === 0) {
    validationErrors.push('Invalid post text provided.');
  }
  if (validationErrors.length > 0) {
    return validationErrors;
  }
  await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify(postData),
    headers: { 'Content-Type': 'application/json' },
  });
  return redirect('/posts');
}
```

```jsx
import { Form, redirect, useActionData } from 'react-router-dom';
function NewPost() {
  const validationErrors = useActionData();
  return (
    <Form method='post' id='post-form'>
      <p>
        <label htmlFor='title'>Title</label>
        <input type='text' id='title' name='title' />
      </p>{' '}
      <p>
        <label htmlFor='text'>Text</label>
        <textarea id='text' name='text' rows={3} />
      </p>
      <ul>
        {validationErrors &&
          validationErrors.map(err => <li key={err}>{err}</li>)}
      </ul>
      <button>Save Post</button>
    </Form>
  );
}
```

## Current Navigation Status

```jsx
function NewPost() {
  const validationErrors = useActionData();
  const navigation = useNavigation();
  const isSubmitting = navigation.state !== 'idle';
  return (
    <Form method='post' id='post-form'>
      <p>
        <label htmlFor='title'>Title</label>
        <input type='text' id='title' name='title' />
      </p>{' '}
      <p>
        <label htmlFor='text'>Text</label>
        <textarea id='text' name='text' rows={3} />
      </p>
      <ul>
        {validationErrors &&
          validationErrors.map(err => <li key={err}>{err}</li>)}
      </ul>
      <button disabled={isSubmitting}>
        {' '}
        {isSubmitting ? 'Saving...' : 'Save Post'}
      </button>
    </Form>
  );
}
```

## Submitting Forms Programmatically

```jsx
import {
  redirect,
  useParams,
  useRouteLoaderData,
  useSubmit,
} from 'react-router-dom';
function PostDetails() {
  const params = useParams();
  const posts = useRouteLoaderData('posts');
  const post = posts.find(post => post.id.toString() === params.id);
  const submit = useSubmit();
  function submitHandler(event) {
    event.preventDefault();
    const proceed = window.confirm('Are you sure?');
    if (proceed) {
      submit({ message: 'Your data, if needed' }, { method: 'delete' });
    }
  }
  return (
    <main id='post-details'>
      <h1>{post.title}</h1>
      <p>{post.body}</p>

      <form onSubmit={submitHandler}>
        <button>Delete</button>
      </form>
    </main>
  );
}
export default PostDetails;
// action must be added to route definition!
export async function action({ request }) {
  const formData = await request.formData();
  console.log(formData.get('message'));
  console.log(request.method);
  return redirect('/posts');
}
```

## Without Causing Page Transition

```jsx
function PostDetails() {
  // ... other code & logic
  const fetcher = useFetcher();
  function likePostHandler() {
    fetcher.submit(null, {
      method: 'post',
      action: `/posts/${post.id}/like`, // targeting an action on another route
    });
  }
  return (
    <button className='icon-btn' onClick={likePostHandler}>
  );
}
```

## Deferring Data Loading

I.e., loading a pgae when certain data are still missing

```jsx
export async function loader() {
  return defer({ posts: getPosts() });
}

async function getPosts() {
  const response = await fetch('https://jsonplaceholder.typicode.com/ posts');
  await wait(3); // utility function, simulating a slow response
  if (!response.ok) {
    throw new Error('Could not fetch posts');
  }
  const data = await response.json();
  return data;
}

function PostsLayout() {
  const data = useLoaderData();
  return (
    <div id='posts-layout'>
      <nav>
        <Suspense fallback={<p>Loading posts...</p>}>
          <Await resolve={data.posts}>
            {loadedPosts => <PostsList posts={loadedPosts} />}
          </Await>
        </Suspense>
      </nav>
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

Certain functions can also be excluded from the `await`, such as:

```jsx
export async function loader() {
  return defer({
    posts: getPosts(), // slow operation => deferred
    userData: await getUserData(), // fast operation => NOT deferred
  });
}
```
