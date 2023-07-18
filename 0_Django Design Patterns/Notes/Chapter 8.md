# Chapter 8

> Note: reference the official Celery documentation for more information on each function

## Why Async?

- Django is synchronous
  - When a client requests a web page, the request reaches Django through a view and passes through multiple layers until the web page is rendered
  - Django is synchronous because the communication will halt until Django's processes execute all the code
  - Slow-blocking tasks such as image processing or complex database queries will lead to slow page loads and should be moved out of the **request-response cycle**
  - Synchronous model will also have difficulty handling events not triggered by web requests, e.g., sending out emails every Friday night

### Problems of Async Code

| Condition           | Explanation                                                                                                                               |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Race condition      | If multiple threads modify the same data, the order in which they're executed may affect the final value, leading to an undetermined data |
| Starvation          | Indefinite waiting of one thread due to other threads flooding in                                                                         |
| Deadlock            | If a thread is waiting for a resource that another thread has locked _and vice versa simultaneously_, the entire thing will get stuck     |
| Debugging challenge | Hard to reproduce                                                                                                                         |
| Order Preservation  | Dependencies between code might not be observed when the execution order varies                                                           |

### Pattern: Endpoint Callback

When used purely as an HTTP callback, it's known as a **WebHook**

1. The client calls a service through a channel (e.g., REST), it also provides an endpoint to notify when the result becomes ready
2. The call returns immediately, but the task might not
3. When the task is completed, the service calls the endpoint to notify the initial sender

> Kinda like how async functions and promises work in React.JS

### Pattern: Publish-Subscribe

1. A listener informs a broker process that a reader is interested in subscribing
2. Once a publisher posts a relevant message, the broker dispatches the message to all subscribers

### Pattern: Polling

Not like votes, per se, but for a client to periodically check a service for any new events.

1. The client calls a service
2. The call returns immediately with new events or the task's status
3. The client waits at a periodic interval and calls step 2 again

## Async Solutions for Django

| Tool            | Explanation                                                               |
| --------------- | ------------------------------------------------------------------------- |
| Celery          | Threads-based model for handling computation outside Django's processes   |
| Asyncio         | Python built-in model for executing multiple tasks within the same thread |
| Django Channels | Real-time queue-like architecture to manage I/O events e.g. WebSockets    |

## Celery

```python
# tasks.py
@shared_task
def fetch_feed(feed_id):
  feed_obj = models.Feed.objects.get(id=feed_id)
  feed_obj.page = retrieve_page(feed_obj.feed_url)
  feed_obj.retrieved = timezone.now()
  feed_obj.save()
```

- Usually mentioned in a file called `tasks.py`
- Looks like a normal Python function except for the `@shared_task` decorator
- A shared task can be used by other apps within the same project

Invoking the task:

```bash
>>> from tasks import fetch_feed
>>> fetch_feed.delay(feed_id=some_feed.id)
```

### How Celery Works

1. One can trigger a Celery task upon a request arrives
2. The Celery task will not immediately finish, but will be immediately be put into (one of) the task queues
3. A worker (separate processes that monitor the task queue and **actually** execute them) process will pick up a task and acknowledge the queue so that the task is removed from the queue
4. The worker executes the task
5. The worker will pick up another task for execution once the task is completed

> If only the side effect (i.e., what happens during the execution of a code block) is needed and the returned result is ignored, a result store is not required; a store is only needed to store the results persistently

**Celery beat process**: a task scheduled to run periodically

### Celery Best Practices

> Quick checklist available at `http://celerytaskchecklist.com`

#### Handling Failure

Retrying automatically:

```python
@shared_task(autoretry_for=(GatewayError,), retry_backoff=60, retry_kwargs={'max_retries': 5}, retry_jitter=True)
def fetch_feed(feed_id):
```

1. `autoretry_for` lists all the exceptions to which Celery should automatically retry
2. `retry_backoff` specifies the initial wait period before the first try; **exponential backoff**: waiting longer and longer for a retry in case of a server overload to give the server more time to recover.
3. `retry_jitter` adds a random number to the waiting period to prevent **thundering herds**, which may render the server unusable if a large number of tasks have the same retry pattern and request a resource simultaneously.

An example of manual retry:

```python
@shared_task(bind=True)
  def fetch_feed(self, feed_id):
  try:
    pass
  except (GatewayError) as exc:
    raise self.retry(exc=exc)
```

#### Idempotent Tasks

**Idempotence**: a mathematical property of a function that assures that, if the same arguments invoke a function, the result will always be the same
**Nullipotent function**: a function that has no side effects
E.g. a task that places a fresh order from the customer is not idempotent, but a task that cancels an existing order is idempotent

(Technically, side effects refer to some state variables being changed outside of a function's local scope, but it can be something as simple as disabling/enabling a button)

#### Avoid Writing to Shared or Global State

> A **race condition**: assume a Celery task _A_ that performs some time-consuming image processing. It picks the ten oldest uploaded images and updates a global counter;
> it reads the counter's values from the database, increments by the number of images processed, and overwrites the counter value; if another similar task, _B_, runs at parallel,
> they may overwrite each other's values byt the end of the task, which could result in corrupted data.

Solution: create a table indexed with hash value (unique identifier), status, and file path of each image:

| Image hash           | Completed at              | Image path       |
| -------------------- | ------------------------- | ---------------- |
| SHA256:fweqf32q33... | 2018-02-09T12:34:56+05:30 | /processed/1.jpg |

Instead of relying on the counter to track the number of processed image, count the number of rows (and allow row overwrites, if more than one process picks up the task),
which could prevent race conditions that result in an incorrect counter value. The `Completed at` column could also reduce the chance of two or more processes processing the
same photo.

Otherwise, if a global numerical counter value is used, _A_ and _B_ will read the counter at the exact same time and overwrite each others' value by the end of their task; in
other words, the final value will be based on who writes in the end, resulting in invalid data.

#### Database Updates

When updating a shared state becomes unavoidable, you can use Python's `F` objects if the database supports row-level locks (_note_: MySQL does not support this)

```python
with transaction.atomic():
  feed = Feed.objects.select_for_update().get(id=id)
  feed.html = sanitize(feed.html)
  feed.save()
```

By using `select_for_update`, the `Feed` object's task queue will be locked until the another task is done; i.e., the query will be waiting or blocked until the lock is freed.

#### Avoid Passing Complex Objects to Tasks

Why not pass objects?

1. Arguments get serialized before passing it to a Celery task, where a large object may clog up the queues.
2. The record might have changed (or even deleted) by the time the task has begun execution, given that Celery is asynchronous by nature.

Solution: always pass a primary key or a lookup value to retrieve the latest value of the object from the database (i.e., don't pass large self-containing objects, but just something that can be looked up in the database).

## Asyncio

Celery runs concurrent tasks out of a process, but one might need to run multiple threads within the same process

In a nutshell [RealPython]:

```python
import asyncio

async def count():
  print("One")
  await asyncio.sleep(1)
  print("Two")

async def main():
  await asyncio.gather(count(), count(), count())

if __name__ == "__main__":
  import time
  s = time.perf_counter()
  asyncio.run(main())
  elapsed = time.perf_counter() - s
  print(f"{__file__} executed in {elapsed:0.2f} seconds.")
```

```shell
$ python3 countasync.py
One
One
One
Two
Two
Two
countasync.py executed in 1.01 seconds.
```

> When each task reaches `asyncio.sleep(1)`, the async process signals the event loop to do something else in the meantime

A **coroutine** is a specialized Python function that can suspend its action before reaching `return` and give control to another coroutine in the meantime.

### Asyncio vs Threads

Reasons why threads aren't popular in Python:

1. Threads must be synchronized while accessing shared resources, otherwise there will be race conditions; although locks can be implemented, starvation and deadlocks are still possible
2. Coroutines are lightweight

Downside of `coroutines` is that the entire code must be written asynchronously, i.e., cannot mix blocking and non-blocking code.

#### Example: Async Web Scraping

```python
import asyncio
import aiohttp
from time import time

sites = [
  "http://news.ycombinator.com/",
  "https://www.yahoo.com/",
  "http://www.aliexpress.com/",
  "http://deelay.me/5000/http://deelay.me/",
]

async def find_size(session, url):
  async with session.get(url) as response:
    page = await response.read()    # [1]
    return len(page)

async def show_size(session, url):
  size = await find_size(session, url)
  print("Read {:8d} chars from {}".format(size, url))

async def main(loop):
  async with aiohttp.ClientSession() as session:
    tasks = []
    for site in sites:    # [2]
      tasks.append(loop.create_task(show_size(session, site)))
    await asyncio.wait(tasks)

if __name__ == '__main__':
  start_time = time()
  loop = asyncio.get_event_loop()
  loop.run_until_complete(main(loop))
  print("Ran in {:6.3f} secs".format(time() - start_time))
```

Once the code reaches `[1]`, the program does not get stuck on the `find_size` function; rather, it goes back to `[2]` and instantiates a new task that starts calculating the size of the
second webpage (through the `for` loop). The program does the same amount of work but saves time as it does not get stuck waiting for a webpage's size to calculate, but
instead opts to start the calculation of the second website's size, and so on.

#### Concurrency vs Parallelism

**Concurrency**: ability to perform other tasks while waiting on the current tasks (e.g., while waiting for something to cook, one's free to do other things) vs. **parallelism**: when two or
more execution engines are performing a task (e.g., when two people are cooking the same dish)

_Global interpreter lock (GIL)_: cannot run more than one thread of the Python interpreter at a time, even in multicore systems

## Django Channels

1. A client (e.g., browser) sends HTTP and WebSocket traffic to an **Asynchronous Server Gateway Interface (ASGI)** (essentially the async version of WSGI servers)
2. HTTP requests are handled synchronously (i.e., the request waits until Django sends back a response)
3. Once a WebSocket connection is established, a browser can send or receive messages; there can be one HTTP and one WebSocket router, where the router
   maps incoming messages to a consumer rather than a view.

> A consumer is an event handler that reacts to events, e.g., sending messages back to the browser; it can be both (a)synchronous, but should not mix the two
