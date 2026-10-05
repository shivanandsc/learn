# How the Event Loop Works Internally, and How It Makes FastAPI Fast

> A beginner-friendly guide. Every code block can be copied and run as-is.
> Related reading: [GIL notes](gil.md) and [sync/async/threading/multiprocessing/asyncio](sync-async-threading-multiprocessing-asyncio.md).

---

## 1. What is the event loop? (the one-line answer)

The **event loop** is a plain `while True:` loop that keeps asking one question: **"which task is ready to run right now?"** It runs that task until it hits an `await`, then moves on to the next ready task.

Think of a **waiter in a restaurant**. A good waiter does not stand at table 1 until the food arrives. They take the order, walk to table 2, take that order, and come back to table 1 when the kitchen rings the bell. One waiter, many tables, no wasted waiting.

```
            ┌───────────────────────────────────────────┐
            │              EVENT LOOP (1 thread)        │
            │                                           │
  tasks ──▶ │  while True:                              │
            │      run every task that is READY         │
            │      wait for I/O or timers (efficiently) │
            │      mark newly-finished waits as READY   │
            └───────────────────────────────────────────┘
```

---

## 2. The problem it solves

Most web code spends its life **waiting**: for a database, another API, a file, or the client's network. While waiting, the CPU does nothing.

```python
# Synchronous: the worker is stuck here for the whole second
data = call_database()      # CPU idle, nobody else is served
```

If one worker handles one request at a time, 1,000 users means 1,000 queued requests. The event loop lets **one thread serve thousands of requests**, because while request A is waiting on the database, the loop serves request B.

---

## 3. The building blocks

| Piece | What it is | Restaurant analogy |
|---|---|---|
| **Coroutine** | a function defined with `async def` that can pause and resume | an order that can be put on hold |
| **`await`** | the pause point: "I'm waiting, run something else" | "waiting for the kitchen" |
| **Task** | a coroutine wrapped so the loop can schedule it | an order slip on the waiter's clipboard |
| **Future** | a placeholder for a result that arrives later | the kitchen's promise ticket |
| **Event loop** | the scheduler that runs tasks | the waiter |
| **Selector** (`epoll`, `kqueue`, ...) | the OS facility that says "this socket now has data" | the kitchen bell |

---

## 4. What is actually inside the loop?

Let's look at a real loop (output measured on Linux, Python 3.12):

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()
    print("loop class:", type(loop).__name__)
    print("selector  :", type(loop._selector).__name__)
    print("ready q   :", type(loop._ready).__name__)
    print("scheduled :", type(loop._scheduled).__name__)

asyncio.run(main())
```

```
loop class: _UnixSelectorEventLoop
selector  : EpollSelector
ready q   : deque
scheduled : list
```

(Those underscore names are private internals, so they may differ on other systems or versions. They are shown only for learning.)

The loop owns exactly three things:

| Structure | Holds | Type |
|---|---|---|
| **Ready queue** | callbacks/tasks that can run **right now** | `deque` (a fast FIFO queue) |
| **Scheduled heap** | timers: "run this at time T" (`sleep`, `call_later`) | min-heap ordered by time (a `list` managed with `heapq`) |
| **Selector** | sockets/files being watched for "readable/writable" events | OS `epoll` / `kqueue` / `select` |

### One iteration of the loop (simplified)

```
 ┌─▶ 1. Compute how long we may sleep:
 │       - 0 if the ready queue is not empty
 │       - else time until the earliest timer
 │
 │   2. Ask the selector: "any I/O events within that time?"
 │       (the thread sleeps INSIDE the OS here, using no CPU)
 │
 │   3. For each I/O event  → move its callback to the ready queue
 │   4. For each expired timer → move its callback to the ready queue
 │
 │   5. Run everything that is in the ready queue, one by one
 │       (each runs until its next `await`)
 └───────────────── repeat ───────────────────────────────────
```

### Proof of the ordering

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()
    order = []
    loop.call_later(0.2, order.append, "later 0.2")
    loop.call_later(0.1, order.append, "later 0.1")
    loop.call_soon(order.append, "soon A")
    loop.call_soon(order.append, "soon B")
    order.append("main before sleep")
    await asyncio.sleep(0.3)
    print(order)

asyncio.run(main())
```

```
['main before sleep', 'soon A', 'soon B', 'later 0.1', 'later 0.2']
```

- `call_soon` goes to the **ready queue** (FIFO, so A then B).
- `call_later` goes to the **timer heap** and runs when its time arrives, soonest first.
- The currently running code (`main`) is never interrupted: it finishes its turn up to the `await`.

---

## 5. How can a function "pause"? Generators underneath

`await` works because coroutines are built on the same mechanism as **generators**: a function that can be suspended in the middle and resumed later with its local variables intact.

You can drive a coroutine by hand and see this:

```python
class Y:
    def __await__(self):
        yield "I-am-yielding-to-the-loop"      # the pause

async def c2():
    await Y()
    return 5

x = c2()
print("send ->", x.send(None))                 # runs until the pause
try:
    x.send(None)                               # resume it
except StopIteration as e:
    print("finished with", e.value)
```

```
send -> I-am-yielding-to-the-loop
finished with 5
```

- The first `send(None)` runs the coroutine **until the pause** and hands back what it yielded.
- The second `send(None)` **resumes** it, it runs to the end, and its `return` value arrives inside `StopIteration`.

That is all the loop does with your coroutines: call `send()`, get told "I'm waiting for X", park the coroutine, and resume it when X is ready.

---

## 6. Build a tiny event loop yourself (about 30 lines)

Nothing teaches this better than writing one. This mini loop supports only `sleep`, but it has the same skeleton as the real thing: a **ready queue** and a **timer heap**.

```python
import time, heapq
from collections import deque

class MiniLoop:
    def __init__(self):
        self.ready = deque()        # tasks that can run now
        self.sleeping = []          # heap of (wake_time, id, task)
        self.n = 0

    def create_task(self, gen):
        self.ready.append(gen)

    def run(self):
        while self.ready or self.sleeping:
            now = time.monotonic()

            # move expired timers to the ready queue
            while self.sleeping and self.sleeping[0][0] <= now:
                _, _, task = heapq.heappop(self.sleeping)
                self.ready.append(task)

            # nothing ready? sleep until the next timer
            if not self.ready:
                time.sleep(max(0, self.sleeping[0][0] - time.monotonic()))
                continue

            # run ONE task until it yields
            task = self.ready.popleft()
            try:
                cmd = next(task)
                if cmd[0] == "sleep":
                    self.n += 1
                    heapq.heappush(self.sleeping,
                                   (time.monotonic() + cmd[1], self.n, task))
            except StopIteration:
                pass                # task finished


def sleep(seconds):
    yield ("sleep", seconds)        # "park me, wake me after N seconds"

def worker(name, delay):
    print(f"  {name} start")
    yield from sleep(delay)
    print(f"  {name} done")


loop = MiniLoop()
for name, delay in [("A", 1.0), ("B", 0.5), ("C", 0.7)]:
    loop.create_task(worker(name, delay))

start = time.perf_counter()
loop.run()
print(f"total: {time.perf_counter() - start:.2f}s")
```

**Output (measured)**

```
  A start
  B start
  C start
  B done
  C done
  A done
total: 1.00s
```

Three tasks of 1.0s, 0.5s and 0.7s finished in **1.00s total** (the longest one), not 2.2s. One thread, no real threads, no parallelism, just clever waiting. The real `asyncio` loop adds a **selector** for network I/O on top of this same idea.

---

## 7. The selector: how the loop waits for network data without wasting CPU

The ready queue and timers handle "run now" and "run later". For network data, the loop asks the **operating system** to watch sockets using `epoll` (Linux), `kqueue` (macOS/BSD) or IOCP (Windows, via a different loop type).

Python exposes this in the `selectors` module:

```python
import selectors, socket, threading, time

sel = selectors.DefaultSelector()
a, b = socket.socketpair()                  # two connected sockets
a.setblocking(False)
sel.register(a, selectors.EVENT_READ, data="socket A")

print("select (timeout 0.2):", sel.select(timeout=0.2))   # nothing yet

# 0.3 seconds from now, "the other side" sends data
threading.Timer(0.3, lambda: b.send(b"hello")).start()

t = time.perf_counter()
for key, mask in sel.select(timeout=5):     # sleeps inside the OS, uses ~0% CPU
    print(f"woke after {time.perf_counter() - t:.1f}s -> "
          f"{key.data} readable, got {key.fileobj.recv(10)}")
```

```
select (timeout 0.2): []
woke after 0.3s -> socket A readable, got b'hello'
```

`sel.select()` blocks **inside the kernel** until something happens. It costs no CPU while waiting, and it can watch **thousands of sockets at once**. This is the engine of every async web server.

### Life of one `await` on the network

```
async def handler():
    data = await client.get(url)       # (1)
    return data                        # (4)

(1) coroutine sends the request, then yields: "wake me when this socket is readable"
(2) loop registers the socket with the selector and moves on to OTHER tasks
(3) data arrives → the OS reports "socket readable" → loop puts the task in the ready queue
(4) loop resumes the coroutine exactly where it paused
```

---

## 8. The golden rule: never block the loop

The loop is **one thread**. If any task runs a long blocking call (`time.sleep`, a slow `requests.get`, a heavy calculation) **without an `await`**, the loop cannot run its next iteration. Every other task, and every other user, **freezes**.

```
Good task:   run ▶ await ......................... (loop serves others) ▶ resume
Bad task:    run ████████████████████████████████ (loop frozen, everyone waits)
```

This single idea explains most async performance bugs, including the FastAPI ones below.

---

## 9. How the event loop helps FastAPI

### 9.1 The stack

FastAPI does not run by itself. It sits on top of an ASGI server and a toolkit:

```
Browser ──HTTP──▶ Uvicorn (ASGI server)  ◀── owns the EVENT LOOP
                      │
                      ▼
                  Starlette (routing, middleware)
                      │
                      ▼
                  FastAPI (validation, dependency injection, docs)
                      │
                      ▼
                  Your endpoint function
```

- **ASGI** is the async standard (the successor of WSGI) that lets a server hand a request to your code as a coroutine.
- **Uvicorn** starts the event loop and keeps thousands of client connections open on it.
- Each incoming request becomes a task on that loop.
- If `uvloop` is installed (included with `pip install "uvicorn[standard]"`), Uvicorn can use it as a faster drop-in replacement for the default loop.

### 9.2 `async def` vs `def` endpoints: the crucial difference

FastAPI runs the two kinds of endpoints differently:

| You write | FastAPI runs it | Runs in |
|---|---|---|
| `async def endpoint()` | directly **on the event loop** | the main (loop) thread |
| `def endpoint()` | in a **thread pool**, so the loop is not blocked | a worker thread (AnyIO, default limit of 40 threads) |

You can prove it:

```python
# app.py
import threading
from fastapi import FastAPI

app = FastAPI()

@app.get("/whoami")
async def whoami():
    return {"thread": threading.current_thread().name}

@app.get("/whoami-sync")
def whoami_sync():
    return {"thread": threading.current_thread().name}
```

```
/whoami       -> {'thread': 'MainThread'}
/whoami-sync  -> {'thread': 'AnyIO worker thread'}
```

(Measured with FastAPI 0.142 and Uvicorn 0.54.)

### 9.3 The experiment: four endpoints, same 1-second "work"

```python
# app.py
import asyncio, time
from fastapi import FastAPI

app = FastAPI()

@app.get("/async-good")             # waits WITHOUT blocking the loop
async def async_good():
    await asyncio.sleep(1)
    return {"endpoint": "async-good"}

@app.get("/async-bad")              # BLOCKS the loop (the classic mistake)
async def async_bad():
    time.sleep(1)
    return {"endpoint": "async-bad"}

@app.get("/sync")                   # plain def: FastAPI uses a thread
def sync_endpoint():
    time.sleep(1)
    return {"endpoint": "sync"}

@app.get("/async-thread")           # async def + blocking work moved to a thread
async def async_thread():
    await asyncio.to_thread(time.sleep, 1)
    return {"endpoint": "async-thread"}

@app.get("/health")
async def health():
    return {"ok": True}
```

Run the server:

```bash
pip install fastapi "uvicorn[standard]" httpx
uvicorn app:app --port 8765
```

Load-test client (fires N requests at once and times them):

```python
# client.py
import asyncio, time, httpx

async def burst(path, n):
    async with httpx.AsyncClient(base_url="http://127.0.0.1:8765", timeout=60) as c:
        start = time.perf_counter()
        await asyncio.gather(*(c.get(path) for _ in range(n)))
        print(f"{path:14s} x{n}: {time.perf_counter() - start:.2f}s")

async def main():
    await burst("/async-good", 10)
    await burst("/async-bad", 5)
    await burst("/sync", 10)
    await burst("/sync", 50)
    await burst("/async-thread", 10)

asyncio.run(main())
```

**Results (measured)**

| Endpoint | Concurrent requests | Total time | What it tells us |
|---|---|---|---|
| `async def` + `await asyncio.sleep(1)` | 10 | **1.02s** | all 10 waits overlap on one thread |
| `async def` + `time.sleep(1)` | 5 | **5.02s** | each request blocked the loop, so they ran one by one |
| `def` + `time.sleep(1)` | 10 | **1.02s** | FastAPI moved them to threads, so they overlap |
| `def` + `time.sleep(1)` | 50 | **2.07s** | thread pool limit is **40**: 40 run first, then the other 10 |
| `async def` + `to_thread(time.sleep)` | 10 | **2.05s** | see the note below |

> **Why 2.05s for `to_thread`?** `asyncio.to_thread` uses asyncio's own default executor, whose size is `min(32, cpu_count + 4)`. My sandbox has 1 core, so that is only **5 threads**, and 10 requests took 2 rounds. On a 4-core machine you would get 8 threads, and so on. You can pass your own executor if you need more. The 40-thread limit for plain `def` endpoints comes from AnyIO, which FastAPI/Starlette use.

### 9.4 The nastiest effect: one bad endpoint hurts everyone

Here is the real danger. While another endpoint is running, how long does a trivial `/health` request take?

```python
# client2.py
import asyncio, time, httpx

async def timed(c, path):
    t = time.perf_counter()
    await c.get(path)
    return time.perf_counter() - t

async def scenario(blocker):
    async with httpx.AsyncClient(base_url="http://127.0.0.1:8765", timeout=60) as c:
        slow = asyncio.create_task(timed(c, blocker))   # start the slow request
        await asyncio.sleep(0.1)
        h = await timed(c, "/health")                   # a different user
        await slow
        print(f"while {blocker:12s} is running, /health took {h*1000:.0f} ms")

async def main():
    await scenario("/async-good")
    await scenario("/sync")
    await scenario("/async-bad")

asyncio.run(main())
```

**Results (measured)**

```
while /async-good  is running, /health took 5 ms
while /sync        is running, /health took 4 ms
while /async-bad   is running, /health took 905 ms
```

With the blocking `async def`, an unrelated health check went from **5 ms to 905 ms**, because the loop was frozen. In production this shows up as random latency spikes, failed load-balancer health checks, and timeouts that are hard to trace to their cause.

---

## 10. Which style should I use in FastAPI?

| What your endpoint does | Use | Why |
|---|---|---|
| Calls an **async library** (`httpx.AsyncClient`, `asyncpg`, async SQLAlchemy, `aiofiles`) | `async def` + `await` | best efficiency, thousands of waits on one thread |
| Calls a **blocking library** (`requests`, classic SQLAlchemy session, `psycopg2`, most SDKs) | plain `def` | FastAPI automatically runs it in a thread |
| Needs to call a blocking function **from inside** an `async def` | `await asyncio.to_thread(func, ...)` (or `run_in_threadpool`) | keeps the loop free |
| Does **heavy CPU work** (image processing, ML inference, big data crunching) | a process pool or a task queue (Celery, RQ, etc.) | neither the loop nor the GIL-bound threads help with CPU parallelism |
| Trivial, instant work (return a constant, tiny computation) | `async def` | no thread-hop cost |

### Scenario examples

**Scenario A: call an external API (async library, ideal)**

```python
import httpx
from fastapi import FastAPI

app = FastAPI()
client = httpx.AsyncClient()

@app.get("/weather")
async def weather():
    r = await client.get("https://api.example.com/weather")   # loop serves others meanwhile
    return r.json()
```

**Scenario B: legacy blocking library (use `def`)**

```python
import requests

@app.get("/legacy")
def legacy():                               # plain def -> runs in the thread pool
    return requests.get("https://api.example.com/data").json()
```

**Scenario C: CPU-heavy work (offload to processes)**

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

pool = ProcessPoolExecutor()

def heavy(n: int) -> int:
    return sum(i * i for i in range(n))

@app.get("/compute/{n}")
async def compute(n: int):
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(pool, heavy, n)   # another process; loop stays free
    return {"result": result}
```

**Scenario D: do something after responding (background task)**

```python
from fastapi import BackgroundTasks

def send_email(address: str):
    ...                                     # slow, blocking work

@app.post("/signup")
async def signup(address: str, background: BackgroundTasks):
    background.add_task(send_email, address)   # runs after the response is sent
    return {"status": "welcome"}
```

---

## 11. Scaling beyond one loop

One event loop runs on **one CPU core**. That is plenty for I/O-bound work, but:

- To use all cores, run **several worker processes**, each with its **own event loop**: `uvicorn app:app --workers 4` (or Gunicorn with Uvicorn workers).
- Workers do not share memory, so keep shared state in a database or cache such as Redis, not in Python globals.
- For CPU-heavy tasks, offload to process pools or a job queue as shown above.

```
        ┌──────── load balancer / OS ────────┐
        ▼            ▼            ▼          ▼
   Worker 1      Worker 2      Worker 3   Worker 4     (4 processes)
   own loop      own loop      own loop   own loop     (each handles thousands
   core 1        core 2        core 3     core 4        of concurrent waits)
```

---

## 12. Common mistakes and fixes

| Mistake | Symptom | Fix |
|---|---|---|
| `time.sleep()` inside `async def` | whole server freezes for that time | `await asyncio.sleep()` |
| `requests.get()` inside `async def` | latency spikes for all users | `httpx.AsyncClient`, or make the endpoint a plain `def` |
| Heavy loop/calculation in `async def` | everything stalls while it runs | process pool or task queue |
| Forgetting `await` | you get a coroutine object, and a `never awaited` warning | add `await` |
| Using `async def` with a sync DB driver | slow under load, blocks the loop | use an async driver, or plain `def` |
| Creating a new `httpx.AsyncClient()` per request | slow (no connection reuse) | create one client and reuse it |
| Keeping state in globals with multiple workers | inconsistent data between workers | external store (Redis, DB) |

---

## 13. Quick summary

- The **event loop** is a single-threaded scheduler built from three parts: a **ready queue**, a **timer heap**, and an OS **selector** (`epoll`/`kqueue`).
- Coroutines **pause** at `await` (generators underneath), and the loop **resumes** them when their I/O or timer is ready.
- While a task waits, the loop runs other tasks, so **one thread can handle thousands of concurrent waits**.
- **Never block the loop.** A single blocking call freezes every request (5 ms became 905 ms in our test).
- **FastAPI + Uvicorn:** each request is a task on the loop. `async def` runs on the loop. Plain `def` runs in a thread pool (default 40 threads).
- Use `async def` with async libraries, plain `def` with blocking libraries, and process pools or queues for CPU-heavy work.
- Scale across cores with multiple worker processes, each with its own loop.

## 14. Cheat sheet

```python
# Pause point (loop serves others while waiting)
await asyncio.sleep(1)
await client.get(url)                       # async HTTP client

# Blocking function inside async code
await asyncio.to_thread(blocking_func, arg)

# CPU-heavy function inside async code
loop = asyncio.get_running_loop()
await loop.run_in_executor(process_pool, cpu_func, arg)

# Run things concurrently
await asyncio.gather(coro1(), coro2())

# Run the server
#   uvicorn app:app --reload                (development)
#   uvicorn app:app --workers 4             (use more cores)
```

```
FastAPI rule of thumb:
  async library  → async def + await
  blocking lib   → def
  CPU heavy      → process pool / queue
```

## 15. Practice exercises

1. Extend `MiniLoop` so a task can also yield `("sleep", 0)` to give up its turn without waiting. Use it to interleave two tasks that print numbers.
2. Reproduce section 9.4 on your own machine. What `/health` latency do you get while `/async-bad` is running?
3. Change `/sync` to take 3 seconds and fire 100 requests at once. Predict the total time from the 40-thread limit, then measure it.
4. Replace `time.sleep(1)` in `/async-bad` with `await asyncio.sleep(1)` and repeat the 5-request test. How does the time change?
5. Add `print(len(asyncio.all_tasks()))` to an `async def` endpoint and fire 100 concurrent requests. How many tasks do you see on the loop?
