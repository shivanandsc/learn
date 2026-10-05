# How Memory Management Works in Python (Internally)

> A beginner-friendly guide. Every code block below can be copied and run as-is.
> All numbers shown were measured on **CPython 3.12.3, 64-bit Linux**. Exact sizes can differ slightly on other versions and systems.
> Related reading: [GIL notes](gil.md) (the GIL exists *because of* reference counting) and [descriptors](descriptor.md).

---

## 1. The big picture (the one-paragraph answer)

Python manages memory **for you**, using three cooperating systems:

1. **Reference counting** frees most objects *instantly*, the moment nothing points to them.
2. A **cyclic garbage collector** cleans up the rare leftovers that reference counting cannot handle (objects pointing at each other in a loop).
3. A **memory allocator** (`pymalloc`) hands out and recycles small chunks of memory efficiently, so Python does not have to ask the operating system for every tiny object.

```
 Your code:   x = [1, 2, 3]
                  │
                  ▼
 ┌────────────────────────────────────────────────────────────┐
 │ 1. REFERENCE COUNTING   "how many names point to this?"    │
 │       count hits 0  ──▶  free it immediately               │
 ├────────────────────────────────────────────────────────────┤
 │ 2. CYCLIC GC            "find loops that count can't free" │
 │       runs now and then, in generations                    │
 ├────────────────────────────────────────────────────────────┤
 │ 3. ALLOCATOR (pymalloc) "arenas → pools → blocks"          │
 │       reuses memory; asks the OS only when needed          │
 └────────────────────────────────────────────────────────────┘
```

> This guide describes **CPython**, the standard Python. Other implementations (PyPy, Jython) manage memory differently.

---

## 2. First idea: variables are labels, objects live elsewhere

In Python, a variable is **not a box that holds a value**. It is a **name tag** attached to an object that lives in memory.

```python
a = [1, 2, 3]
b = a                       # a second tag on the SAME object, no copy!

print(a is b)               # True  (same object)
print(id(a) == id(b))       # True  (same memory identity)

b.append(4)
print(a)                    # [1, 2, 3, 4]   <- changed through the other name
```

```
a ──┐
    ├──▶ [1, 2, 3, 4]      one object, two names
b ──┘
```

To really duplicate, you must ask for a copy:

```python
c = a.copy()
print(c is a, c == a)       # False True   (different object, equal contents)
```

| Operator | Question it answers |
|---|---|
| `==` | do they have **equal values**? |
| `is` | are they the **very same object**? |

### Shallow vs deep copy (a common memory surprise)

```python
import copy

nested = [[1, 2], [3]]
shallow = nested.copy()           # new outer list, SAME inner lists
deep = copy.deepcopy(nested)      # everything duplicated

nested[0].append(99)
print(shallow)                    # [[1, 2, 99], [3]]   <- affected!
print(deep)                       # [[1, 2], [3]]       <- independent
```

### The mutable-default trap

Default values are created **once**, when the function is defined, and then shared by every call:

```python
def add(x, lst=[]):               # the list is created ONCE
    lst.append(x)
    return lst

print(add(1), add(2))             # [1, 2] [1, 2]   <- same list both times!
```

Fix: use `lst=None` and create a new list inside the function.

---

## 3. What is inside every Python object?

Every object in CPython starts with a small **header** written in C:

```c
typedef struct {
    Py_ssize_t ob_refcnt;     // how many references point here (the reference count)
    PyTypeObject *ob_type;    // pointer to its type (int, list, str, ...)
} PyObject;
```

So even the tiniest object carries bookkeeping. Measure it with `sys.getsizeof`:

```python
import sys

print(sys.getsizeof(0))          # 28   an int
print(sys.getsizeof(10**30))     # 40   bigger ints use more
print(sys.getsizeof(""))         # 41   an empty string
print(sys.getsizeof("hello"))    # 46
print(sys.getsizeof([]))         # 56   an empty list
print(sys.getsizeof({}))         # 64   an empty dict
print(sys.getsizeof(object()))   # 16   the bare minimum: just the header
```

> `getsizeof` is **shallow**: it measures the container itself, not the objects inside it. A list of 1,000,000 ints reports only its array of pointers, and the ints add much more on top.

---

## 4. Layer 1: Reference counting

### How it works

Each object keeps a counter of how many references point to it.

- Count **goes up** when you assign it to a new name, put it in a container, or pass it to a function.
- Count **goes down** when a name is deleted or rebound, a container drops it, or a function returns.
- When the count reaches **0**, the object is destroyed **immediately** and its memory is released.

```python
import sys

x = []
print(sys.getrefcount(x))     # 2   (the name `x` + the temporary argument to getrefcount itself)

y = x
print(sys.getrefcount(x))     # 3   (+1: the name `y`)

lst = [x]
print(sys.getrefcount(x))     # 4   (+1: the list holds it)

del y
print(sys.getrefcount(x))     # 3   (-1)

lst.clear()
print(sys.getrefcount(x))     # 2   (-1)
```

> `getrefcount` always reports **1 more** than you expect, because passing the object into the function adds a temporary reference.

### Watching an object die

`__del__` is called right before an object is destroyed, so we can *see* the moment:

```python
class Res:
    def __init__(self, n):
        self.n = n
        print(f"  {n} created")
    def __del__(self):
        print(f"  {self.n} destroyed")

r = Res("A")
r2 = r                          # second reference
del r
print("  after del r (still referenced)")
del r2
print("  after del r2")

Res("temp")                     # nobody keeps it
print("  next line")
```

**Output**

```
  A created
  after del r (still referenced)
  A destroyed
  after del r2
  temp created
  temp destroyed
  next line
```

Key lessons:

- **`del` removes a name, not an object.** `del r` only dropped the count from 2 to 1. The object died when the *last* reference (`r2`) was deleted.
- `Res("temp")` had no name at all, so it was destroyed instantly, before the next line ran.
- This is why `with open(...)` and files "just close themselves" in CPython: the moment the last reference disappears, cleanup runs. Do not rely on it for important resources, though. Use `with`.

### Why reference counting is great, and its cost

| Pros | Cons |
|---|---|
| Memory is freed **immediately and predictably** | Every assignment must update a counter (small overhead) |
| No long "stop the world" pauses | **Cannot free reference cycles** (next section) |
| Simple to reason about | Counter updates are not safe across threads, which is **why the GIL exists** |

### Weak references: point at an object without keeping it alive

A **weak reference** does *not* increase the count:

```python
import weakref, sys

class Obj: pass

o = Obj()
w = weakref.ref(o)              # weak: does not add to the count

print(w() is o, sys.getrefcount(o))   # True 2  (count unchanged by the weakref)
del o
print(w())                      # None  <- the object is gone, the weakref knows
```

Weak references are the tool for caches and parent/child links that must not keep objects alive.

---

## 5. The problem: reference cycles

What if objects point at **each other**?

```python
class Node:
    def __init__(self, n):
        self.n = n
        self.other = None
    def __del__(self):
        print(f"  Node {self.n} freed")

import gc
gc.disable()                    # turn the cycle collector OFF for this experiment

n1, n2 = Node(1), Node(2)
n1.other = n2                   # n1 -> n2
n2.other = n1                   # n2 -> n1   (a cycle!)

del n1, n2                      # delete BOTH names
print("  names deleted, but nothing was freed...")
```

```
      ┌────────┐         ┌────────┐
      │ Node 1 │ ──────▶ │ Node 2 │
      │ rc = 1 │ ◀────── │ rc = 1 │
      └────────┘         └────────┘
   Nobody outside points here, yet each count is 1,
   because they hold each other. Reference counting is stuck.
```

Neither count can reach 0, so neither object is freed. Nothing prints. This would be a **memory leak**. Reference counting alone cannot solve it.

---

## 6. Layer 2: The cyclic garbage collector

The `gc` module's collector exists for exactly this. It only looks at **container objects** (lists, dicts, instances, and so on, the things that can hold references). Ints and strings cannot form cycles, so they are never tracked.

Continuing the experiment:

```python
print("gc.collect() found", gc.collect(), "unreachable objects")
gc.enable()
```

**Output**

```
  Node 1 freed
  Node 2 freed
gc.collect() found 18 unreachable objects
```

Both nodes were freed once the collector ran. (The number found varies, because it also counts other cyclic garbage that happened to exist, such as the nodes' `__dict__`s.)

### How does it find garbage in a cycle?

A simplified version of the algorithm:

1. For each tracked container, copy its reference count into a temporary field `gc_refs`.
2. For each container, look at everything it points to. For every referent that is *also being scanned*, **subtract 1** from the referent's `gc_refs`. This removes references that come from *inside* the group.
3. Any object whose `gc_refs` is still **greater than 0** must be referenced from **outside** (a live variable, say), so it is **reachable**.
4. Everything reachable from those survivors is also marked reachable.
5. All the rest is **unreachable cyclic garbage**, so it is finalized and freed.

```
Node 1 (rc=1)  ──▶  Node 2 (rc=1)
     ▲                    │
     └────────────────────┘

Subtract the internal references:  Node 1 → 0,  Node 2 → 0
Nobody has gc_refs > 0  →  nothing reaches them from outside  →  garbage!
```

### Generations: don't check everything every time

Most objects die young; long-lived ones tend to stay alive. So the collector sorts objects into **generations** and checks young ones often and old ones rarely.

```python
import gc
print(gc.get_threshold())    # (700, 10, 10)   on Python 3.12
print(gc.get_count())        # e.g. (3, 0, 0)  current counters
print(gc.isenabled())        # True
```

```
 Generation 0 (young)     collected most often  ← new objects start here
        │ survivors move up
        ▼
 Generation 1 (middle)    collected less often
        │ survivors move up
        ▼
 Generation 2 (old)       collected rarely      ← long-lived objects
```

With the default `(700, 10, 10)`:

- Generation 0 is collected when (allocations − deallocations) of tracked objects exceeds **700**.
- Generation 1 is collected after generation 0 has been collected **10** times.
- Generation 2 is collected after generation 1 has been collected **10** times.

### Useful `gc` functions

| Function | What it does |
|---|---|
| `gc.collect()` | force a full collection now; returns the number of unreachable objects found |
| `gc.disable()` / `gc.enable()` | turn automatic collection off / on |
| `gc.get_threshold()` / `gc.set_threshold(...)` | read / tune when collections trigger |
| `gc.get_count()` | current counters for each generation |
| `gc.get_stats()` | per-generation statistics |
| `gc.freeze()` | move all current objects to a permanent generation that is never scanned (useful before forking worker processes) |

### The GC has changed between versions

The collector is an area of active change, so check your own version:

| Python version | Cycle collector |
|---|---|
| 3.12 | 3 generations, default thresholds `(700, 10, 10)` |
| 3.13 | 3 generations (defaults changed, e.g. generation 0 threshold is 2000) |
| 3.14.0 to 3.14.4 | a new **incremental** collector |
| 3.14.5 and later | **reverted back** to the 3-generation collector, because the incremental one caused significant memory pressure in production |

A new design that is both generational and incremental has been proposed (PEP 848), but it is a proposal, not something you can rely on today. Run `gc.get_threshold()` on your version, and read the "What's New" notes of your Python release for the current behaviour.

---

## 7. Layer 3: The memory allocator (`pymalloc`)

### The problem

Python creates and destroys **millions of tiny objects** (ints, small strings, tuples). Asking the operating system for memory each time (`malloc`) would be slow and would waste space. So CPython runs its own allocator for **small objects**.

### The three-level structure

```
 Operating system
        │  gives big chunks
        ▼
 ┌──────────────────────── ARENA (1 MiB) ────────────────────────┐
 │  ┌──── POOL (16 KiB) ────┐  ┌──── POOL (16 KiB) ────┐   ...    │
 │  │ block block block ... │  │ block block block ... │          │
 │  │ (all 32 bytes each)   │  │ (all 64 bytes each)   │          │
 │  └───────────────────────┘  └───────────────────────┘          │
 └─────────────────────────────────────────────────────────────────┘
```

| Level | Size (3.12, 64-bit) | Purpose |
|---|---|---|
| **Arena** | 1 MiB | big chunk requested from the OS |
| **Pool** | 16 KiB | a slice of an arena, holding blocks of **one size class** |
| **Block** | 16, 32, 48, ... 512 bytes | the unit handed to an object |

- Requests of **512 bytes or less** go to `pymalloc`. They are rounded up to the nearest size class (there are 32 classes, in steps of 16 bytes).
- Requests **larger than 512 bytes** go straight to the system `malloc`.
- When an object dies, its block goes back to its pool's **free list** and is reused by the next object of that size, with no OS involvement.

### Look at your own allocator

```python
import sys
sys._debugmallocstats()      # prints to stderr; CPython-specific and unofficial
```

A trimmed sample of the real output:

```
Small block threshold = 512, in 32 size classes.

class   size   num pools   blocks in use  avail blocks
-----   ----   ---------   -------------  ------------
    0     16           1              39           982
    1     32           2             621           399
    2     48           8            2696            24
    3     64          20            4895           205
  ...
   31    512           1              22             9

# arenas allocated current         =                    2
2 arenas * 1048576 bytes/arena     =            2,097,152
```

Each row shows how many pools hold blocks of that size and how many blocks are in use or free.

### Why Python does not always give memory back to the OS

An arena can be returned to the OS only when **every pool in it is completely empty**. If even one live object sits in an arena, the whole 1 MiB stays. Over time, scattered survivors cause **fragmentation**, and a long-running process may keep a high memory footprint even after you free lots of objects. The freed memory is not wasted, because Python reuses it for new objects, but your OS's task manager may still show a large number.

> **Practical tip:** if a process must give memory back, process one big batch in a **separate process** (or worker) that exits when finished.

### Free lists

For very common small objects (floats, tuples of small sizes, lists, dicts), CPython also keeps **free lists**: recently freed objects kept ready for instant reuse. You can see them at the end of `_debugmallocstats()` output, for example `free PyFloatObjects`, `free PyTupleObjects`.

---

## 8. Clever tricks for specific types

### 8.1 Small integer cache

Ints from **-5 to 256** are pre-created once and reused everywhere.

```python
a = int("256"); b = int("256"); print(a is b)    # True   (cached)
a = int("257"); b = int("257"); print(a is b)    # False  (two separate objects)
a = int("-5");  b = int("-5");  print(a is b)    # True
a = int("-6");  b = int("-6");  print(a is b)    # False
```

> I used `int("...")` on purpose. If you write `a = 257; b = 257` on one line, the compiler may merge the two identical constants into one object and `is` says `True`, which confuses beginners. **Never use `is` to compare numbers or strings.** Use `==`.

### 8.2 Immortal objects (Python 3.12+)

Objects that live for the whole program, such as `None`, `True`, and the small ints, are now **immortal**: their reference count is a special huge value that never changes, which saves work and helps with multi-threading.

```python
import sys
print(sys.getrefcount(None))   # 4294967295   <- the "immortal" marker (Python 3.12+)
print(sys.getrefcount(5))      # 4294967295
```

### 8.3 String interning

Python stores certain strings (identifiers, short literals) only **once** and reuses them:

```python
s1 = "hello"; s2 = "hello"
print(s1 is s2)                    # True   (the compiler reuses the literal)

s3 = "".join(["hel", "lo"])        # built at runtime
print(s3 == s1, s3 is s1)          # True False   (equal, but a separate object)

import sys
print(sys.intern(s3) is s1)        # True   (intern() makes it share the stored one)
```

`sys.intern` can save memory and speed up comparisons when you have millions of repeated strings.

### 8.4 Lists over-allocate

A list grows by reserving **extra room** ahead of time, so `append` does not reallocate every time:

```python
import sys

L = []
last = -1
for i in range(40):
    size = sys.getsizeof(L)
    if size != last:
        print(f"len={len(L):2d}  size={size} bytes")
        last = size
    L.append(i)
```

```
len= 0  size=56 bytes
len= 1  size=88 bytes
len= 5  size=120 bytes
len= 9  size=184 bytes
len=17  size=248 bytes
len=25  size=312 bytes
len=33  size=376 bytes
```

The size jumps in **steps**, not on every append. This makes `append` fast on average (amortized O(1)) at the price of some unused space.

---

## 9. Measuring memory: finding where it goes

### 9.1 `tracemalloc` (built in)

```python
import tracemalloc

tracemalloc.start()

data = [str(i) * 10 for i in range(100_000)]

current, peak = tracemalloc.get_traced_memory()
print(f"current={current/1e6:.1f}MB  peak={peak/1e6:.1f}MB")
# current=9.8MB  peak=9.8MB

snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:1]:
    print(stat)              # shows the exact file:line that allocated the most
# your_file.py:6: size=9560 KiB, count=100002, average=98 B

del data
current, peak = tracemalloc.get_traced_memory()
print(f"after del: current={current/1e6:.2f}MB  peak={peak/1e6:.1f}MB")
# after del: current=0.00MB  peak=9.8MB   <- reference counting freed it all instantly
```

### 9.2 Other tools

| Tool | Use |
|---|---|
| `sys.getsizeof(obj)` | shallow size of one object |
| `tracemalloc` | which **lines** allocate memory, plus snapshot comparisons |
| `gc.get_objects()` / `gc.get_referrers(obj)` | who is holding on to an object? |
| `objgraph`, `pympler` (third-party) | deep size, reference graphs |
| `memory_profiler` (third-party) | line-by-line memory usage |

---

## 10. Memory leaks in Python (yes, they happen)

Python has no leaks from "forgetting to free", but you can still **keep objects alive by accident**.

### 10.1 An ever-growing cache or global

```python
cache = {}

def process(n):
    cache[n] = [0] * 1000      # stored forever
    return n

for i in range(1000):
    process(i)
# ~8 MB held by the cache and it only grows
```

**Fixes**

```python
import functools

# Option 1: bound the cache size
@functools.lru_cache(maxsize=100)
def process2(n):
    return [0] * 1000

for i in range(1000):
    process2(i)
print(process2.cache_info())    # CacheInfo(hits=0, misses=1000, maxsize=100, currsize=100)
                                # only 100 entries are kept

# Option 2: weak values, so an entry disappears when nobody else uses the object
import weakref

class Big:
    def __init__(self, n): self.n = n

wc = weakref.WeakValueDictionary()
b = Big(1)
wc["k"] = b
print(len(wc))     # 1
del b
print(len(wc))     # 0   <- removed automatically
```

### 10.2 Cycles with `__del__`, or big cycles created in a loop

Modern Python can collect most cycles, even ones with `__del__`. But cycles are collected only when the GC *runs*, so creating huge numbers of them quickly can cause memory to spike. Prefer to **avoid creating cycles**, for example by making the back-reference (child → parent) a `weakref`.

### 10.3 Other usual suspects

| Cause | Fix |
|---|---|
| Global lists/dicts that only append | bound them or clear them |
| Event listeners / callbacks never unregistered | unregister, or store weak references |
| Keeping a traceback or exception object around (it holds the frames and their variables) | do not store `sys.exc_info()` / exceptions long-term |
| Huge lists built just to loop over | use a generator (next section) |
| Large objects held in a long-lived variable | `del big` when done, or restructure into functions |

---

## 11. Writing memory-efficient code

### 11.1 Generators instead of lists

```python
import sys

big = [i for i in range(1_000_000)]       # builds everything in memory
gen = (i for i in range(1_000_000))       # produces one value at a time

print(sys.getsizeof(big))     # 8448728  bytes (just the pointer array, the ints add much more)
print(sys.getsizeof(gen))     # 192      bytes, always tiny

print(sum(i for i in range(1_000_000)))   # 499999500000, computed with almost no memory
```

### 11.2 `__slots__` for classes with millions of instances

Normal instances store their attributes in a per-object dictionary. `__slots__` replaces that with a fixed, compact layout.

```python
import sys

class P1:
    def __init__(self):
        self.x = 1
        self.y = 2

class P2:
    __slots__ = ("x", "y")
    def __init__(self):
        self.x = 1
        self.y = 2

p1, p2 = P1(), P2()
print(sys.getsizeof(p1) + sys.getsizeof(p1.__dict__))   # 344  (object + its dict)
print(sys.getsizeof(p2))                                # 48
print(hasattr(p2, "__dict__"))                          # False
```

The price: you cannot add attributes that are not listed in `__slots__`.

### 11.3 Other habits

- Process big files **line by line** (`for line in f:`) instead of `f.read()`.
- Use `array.array` or NumPy for large numeric data (they store raw numbers, not full Python objects).
- Read data in **chunks** (for example, `pandas.read_csv(..., chunksize=...)`).
- Reuse objects instead of creating new ones inside tight loops.
- `del` big temporary objects when you are done, and keep large work inside functions so locals vanish on return.

---

## 12. A note on free-threaded Python

In the optional **free-threaded build** (no GIL, available from Python 3.13 and officially supported in 3.14), the interpreter can no longer use plain reference counters that only one thread touches at a time. It uses a more complex scheme so threads can safely share objects, and its internals differ from what is described above. For everyday code, the rules of this page still hold (counts reaching zero free objects, a collector for cycles), but some details, such as exact collector behaviour, differ. Check the release notes for your version.

---

## 13. Quick summary

- **Variables are names pointing at objects.** `b = a` copies the *reference*, not the object.
- **Reference counting** is the main mechanism: when an object's count reaches 0 it is freed immediately.
- **`del` removes a name**, not necessarily the object.
- **Cycles** defeat reference counting, so Python also runs a **cyclic garbage collector** that works in **generations** and only tracks container objects.
- The **allocator** serves small objects (512 bytes or less) from **arenas (1 MiB) → pools (16 KiB) → blocks**, reusing freed blocks and sometimes keeping memory from the OS due to fragmentation.
- Special cases: the **small int cache** (-5 to 256), **immortal objects** (3.12+), **string interning**, and **list over-allocation**.
- Measure with `sys.getsizeof` (shallow) and `tracemalloc` (per line).
- "Leaks" in Python are usually **objects you accidentally keep alive**: unbounded caches, globals, callbacks. Fix them with bounded caches, `weakref`, and generators.
- The cycle collector changes between versions (3.14.0 to 3.14.4 had an incremental one, reverted in 3.14.5), so check `gc.get_threshold()` and the release notes for yours.

## 14. Cheat sheet

```python
import sys, gc, weakref, tracemalloc

sys.getrefcount(obj)          # reference count (+1 for the call itself)
sys.getsizeof(obj)            # shallow size in bytes
sys.intern(s)                 # share one copy of a string
sys._debugmallocstats()       # pymalloc arenas/pools/blocks (CPython only)

gc.collect()                  # force cycle collection
gc.get_threshold()            # when collections trigger
gc.get_count()                # current generation counters
gc.disable(); gc.enable()     # turn the cycle collector off/on
gc.get_referrers(obj)         # who points at this object?
gc.freeze()                   # stop scanning current objects (before fork)

w = weakref.ref(obj)          # reference that doesn't keep obj alive
weakref.WeakValueDictionary() # cache that drops unused values

tracemalloc.start()           # begin tracking allocations
tracemalloc.get_traced_memory()      # (current, peak)
tracemalloc.take_snapshot()          # see which lines allocated what
```

```
Object lifecycle:

 create ──▶ refcount = 1 ──▶ names/containers add & remove references
                                   │
              refcount hits 0 ─────┴──▶ freed immediately
              stuck in a cycle ────────▶ freed later by the cyclic GC
```

## 15. Practice exercises

1. Create two objects that reference each other, delete the names, and use `gc.get_objects()` or `gc.collect()` to prove the cycle exists and is then cleaned up. Then rebuild it using `weakref.ref` for one link and show that no `gc.collect()` is needed.
2. Run `sys.getrefcount` on an object while you store it in 1, 2, then 3 different lists. Predict each number before running.
3. Using `tracemalloc`, compare the memory of `[i for i in range(1_000_000)]` versus a generator that sums the same numbers. Report the peak for each.
4. Create 100,000 instances of a normal class and then of a `__slots__` class, and compare the total memory with `tracemalloc`.
5. Write a function with a mutable default argument, show the bug, and then fix it using `None`.
6. Predict the output: `a = int("100"); b = int("100"); print(a is b)` and then the same with `"1000"`. Explain the difference using section 8.1.
7. Build a cache with `WeakValueDictionary`, store an object, delete the last strong reference, and confirm the cache entry disappears.
