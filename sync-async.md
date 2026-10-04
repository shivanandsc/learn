# Sync vs Async in Python: How `threading`, `multiprocessing` and `asyncio` Work

> A beginner-friendly guide with scenario-based examples. Every code block can be copied and run as-is.
> Related reading: [GIL notes](gil.md) (explains *why* threads and processes behave so differently).

---

## 1. The core idea in one picture

Imagine a **restaurant kitchen** with orders (tasks) coming in.

| Style | Kitchen analogy | Python tool |
|---|---|---|
| **Synchronous** | One cook makes one dish start-to-finish, then starts the next | normal code |
| **Threading** | Several cooks share **one kitchen** (one stove, the GIL). They help each other mainly while waiting (water boiling) | `threading` |
| **Multiprocessing** | Several **separate kitchens**, each with its own stove and cook | `multiprocessing` |
| **Asyncio** | **One very organized cook**. While the water boils for dish 1, they start chopping for dish 2, and so on | `asyncio` |

---

## 2. Two kinds of slow work. Identify this first

Choosing the right tool depends on **why** your code is slow.

| Type | What is slow | Examples | CPU busy while waiting? |
|---|---|---|---|
| **I/O-bound** | waiting for something outside the CPU | web requests, DB queries, file/network reads, `sleep` | **No**, the CPU is idle |
| **CPU-bound** | the computation itself | number crunching, image/video processing, compression, ML training | **Yes**, the CPU is at 100% |

> **Golden rule:** I/O-bound: use **threads or asyncio**. CPU-bound: use **multiprocessing**.

---

## 3. Synchronous (sync) code: the baseline

Sync means each line **waits** for the previous one to finish.

```python
import time

def fetch(i):
    time.sleep(1)                 # pretend this is a 1-second network call
    return f"page{i}"

start = time.perf_counter()
results = [fetch(i) for i in range(5)]
print(f"Sync: {time.perf_counter() - start:.2f}s")
```

**Output (measured):** `Sync: 5.00s`

Five 1-second waits happen **one after another**. The CPU sat idle for almost all of that time. This is the waste the other tools remove.

---

## 4. Threading

### How it works

A **thread** is a separate flow of execution **inside the same process**. All threads share the same memory. In standard CPython the **GIL** lets only one thread run Python bytecode at a time, but a thread **releases the GIL while waiting for I/O**, so other threads can run.

```
Thread 1:  run ─▶ [waiting for network.........] ─▶ run
Thread 2:       run ─▶ [waiting for network.........] ─▶ run
Thread 3:            run ─▶ [waiting for network.........] ─▶ run
                     (all the waiting overlaps)
```

### Scenario: download 5 pages

```python
import time
from concurrent.futures import ThreadPoolExecutor

def fetch(i):
    time.sleep(1)
    return f"page{i}"

start = time.perf_counter()
with ThreadPoolExecutor(max_workers=5) as ex:
    results = list(ex.map(fetch, range(5)))
print(f"Threads: {time.perf_counter() - start:.2f}s")
print(results)
```

**Output (measured):** `Threads: 1.00s` (5x faster than sync)

### The danger: shared memory means race conditions

Because threads share variables, two threads can overwrite each other's updates. Protect shared data with a `Lock`:

```python
import threading

counter = 0
lock = threading.Lock()

def add():
    global counter
    for _ in range(100_000):
        with lock:                 # only one thread at a time in here
            counter += 1

threads = [threading.Thread(target=add) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)                     # 400000
```

For passing work between threads, a thread-safe `queue.Queue` is often cleaner than shared variables:

```python
import threading, queue

q = queue.Queue()
results = []

def producer():
    for i in range(3):
        q.put(i)
    q.put(None)                    # signal "no more work"

def consumer():
    while (item := q.get()) is not None:
        results.append(item * 10)

a = threading.Thread(target=producer)
b = threading.Thread(target=consumer)
a.start(); b.start(); a.join(); b.join()
print(results)                     # [0, 10, 20]
```

**Use threads when:** the work is I/O-bound, you use blocking libraries (`requests`, most DB drivers), and you have a moderate number of tasks (tens to hundreds).

---

## 5. Multiprocessing

### How it works

A **process** is a completely separate Python interpreter with its **own memory and its own GIL**. Processes run in **true parallel** on different CPU cores.

```
Core 1:  Process 1  ████████████████
Core 2:  Process 2  ████████████████     <- truly simultaneous
Core 3:  Process 3  ████████████████
```

### Scenario: heavy calculation on 4 jobs

```python
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu(n):
    s = 0
    for i in range(n):
        s += i * i
    return s

N = 3_000_000

if __name__ == "__main__":             # REQUIRED for multiprocessing
    start = time.perf_counter()
    [cpu(N) for _ in range(4)]
    print(f"Sync      : {time.perf_counter() - start:.2f}s")

    start = time.perf_counter()
    with ThreadPoolExecutor(4) as ex:
        list(ex.map(cpu, [N] * 4))
    print(f"Threads   : {time.perf_counter() - start:.2f}s")

    start = time.perf_counter()
    with ProcessPoolExecutor(4) as ex:
        list(ex.map(cpu, [N] * 4))
    print(f"Processes : {time.perf_counter() - start:.2f}s")
```

**What to expect on a machine with 4 or more cores**

```
Sync      : ~0.65s
Threads   : ~0.65s (or slower)   <- GIL: no gain
Processes : ~0.2s                <- close to 4x faster
```

> **Honest note on these numbers:** I ran this script in a sandbox that has only **1 CPU core**, where all three took about 0.65s (processes cannot beat the clock when there is one core). The "expected" figures above are what you should see on a multi-core computer. Run it yourself to see the real difference, and check your core count with `import os; print(os.cpu_count())`.

### Memory is NOT shared

Each process gets its own copy of variables, so a change in the child is invisible to the parent:

```python
import multiprocessing as mp

counter = 0

def inc():
    global counter
    counter += 1

if __name__ == "__main__":
    p = mp.Process(target=inc)
    p.start(); p.join()
    print(counter)       # 0   <- the parent's counter did not change!
```

To share results, use explicit tools:

```python
import multiprocessing as mp

def worker(q, n):
    q.put(n * n)

if __name__ == "__main__":
    q = mp.Queue()
    procs = [mp.Process(target=worker, args=(q, i)) for i in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
    print(sorted(q.get() for _ in range(4)))    # [0, 1, 4, 9]
```

(`mp.Value`, `mp.Array` and `mp.Manager` are the other options for shared state, and they need locks too.)

### The costs

- **Startup is heavier** than a thread (a new interpreter is launched).
- **Data is copied** between processes (it is pickled), so sending huge objects back and forth is slow.
- **Higher memory use.**
- The target function and its arguments must be **picklable** (no lambdas, for example).

**Use multiprocessing when:** the work is CPU-bound and you want to use all your cores.

---

## 6. Asyncio

### How it works

`asyncio` runs **everything in one thread** using an **event loop**. Your functions (called *coroutines*) are written with `async def`, and they voluntarily say "I'm waiting, someone else can run" using `await`.

```
Event loop (single thread):

 task A: run ─▶ await (waiting)...............────▶ resume ─▶ done
 task B:            run ─▶ await (waiting)......────▶ resume ─▶ done
 task C:                       run ─▶ await (waiting)....────▶ resume
```

This is **cooperative multitasking**: tasks switch **only at `await` points**, not at random times as threads do.

### The 3 keywords you must know

| Keyword | Meaning |
|---|---|
| `async def` | defines a coroutine (a function that can pause) |
| `await` | pause here until the thing finishes, and let other tasks run meanwhile |
| `asyncio.run(main())` | start the event loop and run your top-level coroutine |

### Scenario: download 5 pages

```python
import asyncio, time

async def fetch(i):
    await asyncio.sleep(1)            # non-blocking wait (use an async library in real code)
    return f"page{i}"

async def main():
    results = await asyncio.gather(*(fetch(i) for i in range(5)))
    print(results)

start = time.perf_counter()
asyncio.run(main())
print(f"Asyncio: {time.perf_counter() - start:.2f}s")
```

**Output (measured):** `Asyncio: 1.00s`

### Watch the order of execution

```python
import asyncio

async def worker(name, delay):
    print(f"  {name} start")
    await asyncio.sleep(delay)
    print(f"  {name} done")
    return name

async def main():
    print(await asyncio.gather(worker("A", 2), worker("B", 1), worker("C", 1.5)))

asyncio.run(main())
```

**Output**

```
  A start
  B start
  C start
  B done
  C done
  A done
['A', 'B', 'C']
```

All three start immediately, they finish in the order of their delays (B, C, A), and `gather` still returns results in the **original order** (A, B, C).

### Running a task in the background with `create_task`

```python
import asyncio

async def job(n):
    await asyncio.sleep(0.1)
    return n

async def main():
    t1 = asyncio.create_task(job(1))      # starts running right away
    t2 = asyncio.create_task(job(2))
    print("tasks started, doing other work")
    print(await t1, await t2)             # 1 2

asyncio.run(main())
```

### Timeouts and limiting concurrency

```python
import asyncio

# 1) Timeout
async def slow():
    await asyncio.sleep(5)

async def with_timeout():
    try:
        await asyncio.wait_for(slow(), timeout=1)
    except asyncio.TimeoutError:
        print("timed out after 1s")

asyncio.run(with_timeout())

# 2) Limit how many run at once (e.g. be polite to a server)
async def limited(sem, i):
    async with sem:
        await asyncio.sleep(1)
        return i

async def main():
    sem = asyncio.Semaphore(2)            # max 2 at a time
    await asyncio.gather(*(limited(sem, i) for i in range(6)))

asyncio.run(main())                       # takes ~3s: 6 tasks / 2 at a time
```

**Measured:** 6 tasks with a limit of 2 took `3.00s`.

### The #1 asyncio mistake: blocking the event loop

Inside `async def`, a **blocking** call such as `time.sleep()` freezes the **entire** loop, so nothing else can run:

```python
import asyncio, time

async def bad(i):
    time.sleep(1)               # BLOCKS everything
    return i

async def good(i):
    await asyncio.sleep(1)      # lets others run
    return i

async def run(fn):
    await asyncio.gather(*(fn(i) for i in range(3)))

start = time.perf_counter(); asyncio.run(run(bad))
print(f"blocking in async: {time.perf_counter() - start:.2f}s")    # 3.00s

start = time.perf_counter(); asyncio.run(run(good))
print(f"proper await     : {time.perf_counter() - start:.2f}s")    # 1.00s
```

Same code shape, three times slower, because `time.sleep` never gave control back.

### What if I must call blocking code from async code?

Use `asyncio.to_thread` to run it in a thread without blocking the loop:

```python
import asyncio, time

def blocking_io(i):
    time.sleep(1)               # e.g. an old library with no async version
    return i

async def main():
    results = await asyncio.gather(*(asyncio.to_thread(blocking_io, i) for i in range(3)))
    print(results)              # [0, 1, 2]  in ~1s

asyncio.run(main())
```

For CPU-heavy functions, use `loop.run_in_executor` with a `ProcessPoolExecutor` (see section 8).

**Use asyncio when:** you have lots of I/O waits (hundreds to thousands of connections) and the libraries you use have async versions (`aiohttp`, `httpx`, `asyncpg`, and so on).

---

## 7. Threads vs asyncio at scale

For 5 tasks they tie. The difference shows at large scale because a thread costs memory (typically about 8 MB of address space reserved for its stack) and OS scheduling, while a coroutine is just a small Python object.

Measured with 1000 tasks that each wait 1 second:

```
1000 asyncio tasks : 1.01s
1000 threads       : 1.10s   (and far more memory)
```

Both finish quickly here, but at 10,000 or more connections threads become heavy, while asyncio stays light.

---

## 8. Scenario guide: which tool for which job?

| # | Scenario | Bottleneck | Best tool | Why |
|---|---|---|---|---|
| 1 | Download 50 files / call 50 APIs | I/O | `ThreadPoolExecutor` or `asyncio` | waits overlap |
| 2 | Web server handling 10,000 connections | I/O | `asyncio` | one thread, very light per connection |
| 3 | Resize 1,000 images | CPU | `ProcessPoolExecutor` | uses all cores |
| 4 | Scrape pages, then parse heavy HTML | I/O then CPU | asyncio + process pool | each stage gets the right tool |
| 5 | Old library without async support, many calls | I/O | `ThreadPoolExecutor` or `asyncio.to_thread` | blocking code in threads |
| 6 | Background logging or a heartbeat in a GUI app | I/O | `threading` | simple, runs alongside |
| 7 | Train a model on many data chunks | CPU | `multiprocessing` | true parallelism |
| 8 | Simple script, a few quick steps | none | plain sync code | simplest is best |

### Scenario 4 in code: combine asyncio and processes

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu(n):                             # heavy work
    s = 0
    for i in range(n):
        s += i * i
    return s

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor(2) as pool:
        results = await asyncio.gather(
            loop.run_in_executor(pool, cpu, 3_000_000),   # runs in another process
            loop.run_in_executor(pool, cpu, 3_000_000),
            asyncio.sleep(1),                              # I/O wait at the same time
        )
    print(results[:2])

if __name__ == "__main__":
    asyncio.run(main())
```

The event loop stays free to handle I/O while processes crunch numbers.

---

## 9. Full comparison

| | **Sync** | **threading** | **multiprocessing** | **asyncio** |
|---|---|---|---|---|
| Runs in | 1 thread | many threads, 1 process | many processes | 1 thread, 1 process |
| True parallelism (CPU) | No | **No** (GIL) | **Yes** | **No** |
| Good for I/O-bound | No | **Yes** | OK but heavy | **Yes, best at scale** |
| Good for CPU-bound | No | No | **Yes** | No |
| Switching happens | n/a | OS decides, any time | OS decides | only at `await` |
| Memory shared? | n/a | **Yes** (needs locks) | **No** (needs queues/pipes) | Yes, same thread |
| Race conditions? | No | **Yes** | on shared Values only | Rare (only across `await`) |
| Overhead per task | none | medium (~MBs) | high (a whole interpreter) | **very low** |
| Code changes needed | none | little | pickling, `__main__` guard | **`async`/`await` everywhere** |
| Needs async-aware libraries? | no | no | no | **Yes** |
| Debugging | easiest | harder | harder | medium |

## 10. A simple decision flow

```
Is your program slow?
 │
 ├─ No ──▶ Keep it synchronous.
 │
 └─ Yes: why?
      │
      ├─ Waiting on network / disk / DB  (I/O-bound)
      │     │
      │     ├─ Have async libraries and many connections? ─▶ asyncio
      │     └─ Using normal blocking libraries?           ─▶ ThreadPoolExecutor
      │
      └─ Heavy computation (CPU-bound)
            │
            ├─ Python loops?               ─▶ ProcessPoolExecutor / multiprocessing
            └─ NumPy / C-library heavy?    ─▶ threads may already work (they release the GIL)
```

---

## 11. Quick summary

- **Sync:** one thing at a time. Simple, but wastes time when waiting.
- **Threading:** many threads, one process. Great for I/O-bound work, no CPU speedup because of the GIL, and shared memory needs **locks**.
- **Multiprocessing:** many processes, each with its own GIL. The only standard way to get true CPU parallelism. Memory is **not shared**, so use queues, and expect higher overhead.
- **Asyncio:** one thread plus an event loop and cooperative `await`. The best choice for huge numbers of I/O tasks. **Never block the loop**.
- Always ask first: is it **I/O-bound** or **CPU-bound**?
- These tools can be **combined** (asyncio + `to_thread`, asyncio + process pool).
- In Python 3.14, an optional free-threaded build (no GIL) is officially supported, which may change the CPU-bound advice for threads in the future. The default build still has the GIL, so check the current release notes.

## 12. Cheat sheet

```python
# Threads (I/O-bound)
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=10) as ex:
    results = list(ex.map(func, items))

# Processes (CPU-bound)
from concurrent.futures import ProcessPoolExecutor
if __name__ == "__main__":
    with ProcessPoolExecutor() as ex:
        results = list(ex.map(func, items))

# Asyncio (many I/O waits)
import asyncio
async def main():
    return await asyncio.gather(*(coro(x) for x in items))
asyncio.run(main())

# Blocking code inside asyncio
await asyncio.to_thread(blocking_func, arg)

# Limit concurrency / add timeout
sem = asyncio.Semaphore(10)
await asyncio.wait_for(coro(), timeout=5)
```

## 13. Practice exercises

1. Write a function that "downloads" 10 URLs using `time.sleep(0.5)`. Time it as sync, with `ThreadPoolExecutor`, and with `asyncio`. Compare the three times.
2. Change the `bad` async example so it still uses `time.sleep` but is fixed with `asyncio.to_thread`. Measure the time.
3. Run the CPU-bound script on your own computer. How does the process speedup compare with your core count (`os.cpu_count()`)?
4. In the multiprocessing "memory is not shared" example, make the parent see the result by using `mp.Value` with a lock.
5. Build a scraper that fetches 20 pages with asyncio but allows only 5 requests at once (hint: `Semaphore`).
