# How the GIL Works in Python (and Why It Matters)

> A beginner-friendly guide. Every code block below can be copied and run as-is.

---

## 1. What is the GIL? (the one-line answer)

The **Global Interpreter Lock (GIL)** is a lock inside CPython (the standard Python) that lets **only one thread run Python bytecode at a time**, even on a computer with many CPU cores.

Think of a kitchen with 4 cooks (threads) but only **one knife** (the GIL). Only the cook holding the knife can chop. The others wait their turn.

```
Thread 1:  ██████░░░░░░██████░░░░░░
Thread 2:  ░░░░░░██████░░░░░░██████
              ▲ only one runs at any moment
```

> **Important:** the GIL belongs to **CPython**, the reference implementation. It is not part of the Python language itself. Other implementations (Jython, IronPython) do not have it.

---

## 2. Why does the GIL exist?

The reason is **memory management**. CPython tracks how many references point to each object (the *reference count*). When the count reaches zero, the object is freed.

```python
import sys

a = []
print(sys.getrefcount(a))   # 2  (the variable `a` + the temporary argument to getrefcount)

b = a
print(sys.getrefcount(a))   # 3  (now `b` points to it as well)
```

Now imagine two threads changing the same count at the same moment. The count could get corrupted, which leads to:

- **memory leaks** (count too high, object never freed), or
- **crashes** (count too low, object freed while still in use).

There are two ways to protect the count:

| Approach | Cost |
|---|---|
| A lock on **every single object** | slow, and risk of deadlocks |
| **One big lock** for the whole interpreter (the GIL) | simple, fast for single-threaded code, easy for C extensions |

CPython chose the second. It made the interpreter simple and fast when running a single thread, and it made writing C extension modules much easier. That decision is the GIL.

---

## 3. How does the GIL work internally?

A thread must **hold the GIL** to run Python bytecode. The rules are:

1. A thread **acquires** the GIL and starts running bytecode.
2. After a short time slice, the interpreter asks it to **release** the GIL so others get a turn.
3. A thread also **releases** the GIL whenever it **waits for I/O** (network, disk, `time.sleep`), because it is not using the CPU anyway.
4. Another waiting thread then grabs the GIL.

```
Thread A acquires GIL ─▶ runs bytecode ─▶ time slice ends ─▶ releases GIL
                                                                  │
Thread B waits ───────────────────────────────────────────────────▼
                                                      acquires GIL ─▶ runs ...
```

### The switch interval

Since Python 3.2, a waiting thread sets a flag asking the running thread to give up the GIL after a fixed interval. You can inspect it:

```python
import sys
print(sys.getswitchinterval())   # 0.005  (5 milliseconds)
```

So threads take turns roughly every 5 ms. A thread can also lose the GIL earlier if it hits a blocking I/O call.

### Where is the lock checked?

The interpreter loop only checks "should I release the GIL?" **between bytecode instructions**. You can see bytecode with `dis`:

```python
import dis

def inc():
    global c
    c += 1

dis.dis(inc)
```

```
LOAD_GLOBAL   c
LOAD_CONST    1
BINARY_OP     +=
STORE_GLOBAL  c
```

`c += 1` is **four** instructions, not one. A thread switch can happen between them. This matters a lot in section 7.

---

## 4. The significance: what the GIL means for you

| Kind of work | Do threads help? | Why |
|---|---|---|
| **I/O-bound** (web requests, file reads, DB calls, `sleep`) | **Yes, a lot** | the GIL is released while waiting, so other threads run |
| **CPU-bound** (math loops, image processing, parsing) | **No** | only one thread computes at a time, so there is no speedup |
| **CPU-bound with C libraries** (NumPy, many parts of pandas) | **Often yes** | those libraries release the GIL inside their C code |

Summary: **threads give concurrency, not CPU parallelism**, in standard CPython.

---

## 5. Demo 1: I/O-bound work. Threads help

```python
import threading, time

def io_task():
    time.sleep(1)           # simulates waiting for a network call

# Sequential
start = time.perf_counter()
for _ in range(4):
    io_task()
print(f"Sequential: {time.perf_counter() - start:.2f}s")

# Threaded
start = time.perf_counter()
threads = [threading.Thread(target=io_task) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"4 threads : {time.perf_counter() - start:.2f}s")
```

**Output (measured)**

```
Sequential: 4.00s
4 threads : 1.00s
```

All four threads wait at the same time because `sleep` releases the GIL. About 4x faster.

---

## 6. Demo 2: CPU-bound work. Threads do not help, processes do

```python
import threading, time
from concurrent.futures import ProcessPoolExecutor

def count_down(n):
    while n > 0:
        n -= 1

N = 20_000_000

if __name__ == "__main__":
    # 1. Single thread, run twice
    start = time.perf_counter()
    count_down(N); count_down(N)
    print(f"Single thread, 2 jobs : {time.perf_counter() - start:.2f}s")

    # 2. Two threads
    start = time.perf_counter()
    threads = [threading.Thread(target=count_down, args=(N,)) for _ in range(2)]
    for t in threads: t.start()
    for t in threads: t.join()
    print(f"2 threads             : {time.perf_counter() - start:.2f}s")

    # 3. Two processes
    start = time.perf_counter()
    with ProcessPoolExecutor(2) as ex:
        list(ex.map(count_down, [N, N]))
    print(f"2 processes           : {time.perf_counter() - start:.2f}s")
```

**What to expect on a machine with 2 or more cores**

```
Single thread, 2 jobs : ~1.1s
2 threads             : ~1.1s or slower   <- no speedup, the GIL serializes them
2 processes           : ~0.6s             <- roughly 2x faster, each process has its own GIL
```

> **Note:** the exact times depend on your machine. When I ran this script in a sandbox with a single CPU core, single thread and 2 threads both took about 1.1s, but the processes could not show a speedup because there was only one core to use. Run it on your own multi-core computer to see the real difference.

**Why processes work:** each process has its **own interpreter and its own GIL**, so they truly run in parallel. The cost is higher memory use and the need to pass data between processes.

---

## 7. Demo 3: the GIL does NOT make your code thread-safe

A very common myth: "I have the GIL, so I don't need locks." This is **wrong**. The GIL protects the interpreter's internals, not your program's logic.

Recall that `c += 1` is several steps (read, add, write). If a switch happens in the middle, updates get lost:

```python
import threading, time

c = 0

def work():
    global c
    for _ in range(1000):
        tmp = c              # read
        time.sleep(0)        # lets another thread run right here (forces the problem)
        c = tmp + 1          # write (may overwrite another thread's update)

threads = [threading.Thread(target=work) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()

print("unsafe:", c, "(expected 4000)")
```

**Output (measured)**

```
unsafe: 1002 (expected 4000)
```

Most of the updates were lost. The fix is a lock:

```python
import threading, time

c = 0
lock = threading.Lock()

def safe_work():
    global c
    for _ in range(1000):
        with lock:                # only one thread inside at a time
            tmp = c
            time.sleep(0)
            c = tmp + 1

threads = [threading.Thread(target=safe_work) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()

print("safe:", c)                 # 4000
```

> **Why the `time.sleep(0)`?** Without it, the race is rare in modern CPython because the interpreter switches threads only at certain points, so a plain `c += 1` loop often *appears* to work. That is exactly what makes this bug dangerous: it can hide for months and then fail in production. The `sleep(0)` just makes the problem easy to see. The bug exists either way, so always use a `Lock` (or a thread-safe structure such as `queue.Queue`) when threads modify shared data.

---

## 8. Ways to work around the GIL

| Goal | Solution |
|---|---|
| Many I/O waits (web, files, DB) | `threading` or `concurrent.futures.ThreadPoolExecutor` |
| Many I/O waits, thousands of connections | `asyncio` (single thread, very light) |
| Heavy CPU work, use all cores | `multiprocessing` or `ProcessPoolExecutor` |
| Heavy numeric work | NumPy, pandas and similar, whose C code releases the GIL |
| Critical hot loops | write a C, Cython or Rust extension that releases the GIL |
| Run Python threads in true parallel | the **free-threaded build** (section 9) |

Quick example of a thread pool and process pool, which share the same interface:

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def square(x):
    return x * x

if __name__ == "__main__":
    with ThreadPoolExecutor() as ex:       # good for I/O-bound
        print(list(ex.map(square, range(5))))

    with ProcessPoolExecutor() as ex:      # good for CPU-bound
        print(list(ex.map(square, range(5))))
```

---

## 9. The future: Python without the GIL

Python is now moving toward removing the GIL, as described in **PEP 703**.

| Phase | Meaning | Status |
|---|---|---|
| Phase I | free-threaded build available but **experimental** | Python 3.13 |
| Phase II | free-threaded build **officially supported**, but still optional | **Python 3.14** (PEP 779) |
| Phase III | free-threaded build becomes the **default** | not decided yet |

Key facts:

- The **default** Python build still has the GIL. The free-threaded build is a separate, optional build (often called `python3.14t`).
- Single-threaded code runs a little slower on the free-threaded build. The overhead was reported to be roughly 5 to 10% in 3.14.
- Free-threaded builds use their own wheel tag (`cp314t`), so libraries must be built for it.
- Even without the GIL, **you still need locks** for shared data (see section 7).

Check which one you are running:

```python
import sys

# True on a normal build, False on a free-threaded build with the GIL disabled
print(sys._is_gil_enabled())      # available from Python 3.13
```

> Things are changing fast, so check the official Python release notes for the current status.

---

## 10. Quick summary

- The **GIL** is a lock in CPython that allows only **one thread to execute Python bytecode at a time**.
- It exists to protect **reference counting** and keep the interpreter simple and fast for single-threaded code.
- A thread releases the GIL after a **switch interval (5 ms)** or when it **waits for I/O**.
- **I/O-bound** tasks: threads (or `asyncio`) help a lot. **CPU-bound** tasks: threads do not help, so use **multiprocessing**.
- The GIL does **not** make your code thread-safe, so still use locks for shared data.
- Python 3.14 has an officially supported **free-threaded build** with no GIL, but it is optional and the default build still has the GIL.

## 11. Cheat sheet

```python
import sys, threading
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

sys.getswitchinterval()        # how often threads may switch (0.005 s)
sys._is_gil_enabled()          # is the GIL on? (3.13+)

lock = threading.Lock()
with lock:                     # protect shared data
    ...

ThreadPoolExecutor()           # I/O-bound work
ProcessPoolExecutor()          # CPU-bound work
```

## 12. Practice exercises

1. Write a function that downloads 5 URLs one by one, then rewrite it with `ThreadPoolExecutor`. Compare the times.
2. Run Demo 2 on your own computer and record the three timings. How close is the process version to a 2x speedup?
3. In Demo 3, remove `time.sleep(0)` and run it several times. Does the bug appear? Does that mean the code is safe?
4. Use `dis.dis()` on a function containing `x = x + 1` and `lst.append(1)`. Which one is a single instruction, and why does that not automatically make it a good idea to skip locks?
