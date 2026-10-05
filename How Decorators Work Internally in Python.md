# How Decorators Work Internally in Python

> A beginner-friendly guide. Every code block below can be copied and run as-is.
> Related reading: [descriptors](descriptor.md) (explains why decorators work on methods).

---

## 1. What is a decorator? (the one-line answer)

A **decorator** is a function that **takes another function, adds some extra behaviour around it, and returns the upgraded function**, without you editing the original function's code.

Think of **gift wrapping**: the gift (your function) stays exactly the same, but the wrapping (the decorator) adds something around it, such as a ribbon, a card, or a tracking label.

```python
@timer                 # <- the decorator
def slow():            # <- the original function (the gift)
    ...
```

---

## 2. The big secret: `@decorator` is just shorthand

This:

```python
@decorator
def func():
    ...
```

is **exactly** the same as this:

```python
def func():
    ...

func = decorator(func)
```

That is the whole trick. Python defines `func`, passes it to `decorator`, and **rebinds the name `func`** to whatever the decorator returns. Everything else in this page is just consequences of that one line.

---

## 3. Three ideas you need first

### 3.1 Functions are objects (first-class)

You can store a function in a variable, pass it to another function, and return it from a function.

```python
def greet(name):
    return f"Hi {name}"

f = greet                      # store in a variable
print(f("A"), f is greet)      # Hi A True


def call_twice(fn, x):         # pass a function as an argument
    return fn(fn(x))

print(call_twice(lambda s: s + "!", "go"))    # go!!


def make_adder(n):             # return a function from a function
    def add(x):
        return x + n
    return add

add5 = make_adder(5)
print(add5(10))                # 15
```

### 3.2 Closures: inner functions remember

Look at `add5` above. It still knows `n = 5` even though `make_adder` has already finished. That is a **closure**: the inner function carries a hidden reference to the variables it uses from the outer function.

You can see this memory inside the function object:

```python
print(add5.__code__.co_freevars)            # ('n',)
print(add5.__closure__)                     # (<cell at 0x...: int object at 0x...>,)
print(add5.__closure__[0].cell_contents)    # 5
```

A decorator relies on this: the inner `wrapper` function remembers the original function `fn` through a closure.

### 3.3 `*args` and `**kwargs`: accept anything

The wrapper does not know what arguments the original function takes, so it accepts everything and passes it along:

```python
def wrapper(*args, **kwargs):
    return fn(*args, **kwargs)
```

---

## 4. Build your first decorator, step by step

Goal: make a function's result UPPERCASE.

```python
def shout(fn):                       # 1. receives the original function
    def wrapper(*args, **kwargs):    # 2. defines a new function around it
        result = fn(*args, **kwargs) # 3. calls the original (fn is remembered by closure)
        return result.upper()        # 4. adds the extra behaviour
    return wrapper                   # 5. returns the new function (NOT calling it)


def hello(name):
    return f"hello {name}"

hello = shout(hello)                 # the manual way
print(hello("bob"))                  # HELLO BOB


@shout                               # the @ way (identical result)
def bye(name):
    return f"bye {name}"

print(bye("amy"))                    # BYE AMY
```

### What `hello` is after decoration

After `hello = shout(hello)`, the name `hello` points to **`wrapper`**, not to the original function. The original function is still alive, kept by the closure.

```
BEFORE:   hello ──▶ [original hello function]

AFTER:    hello ──▶ [wrapper] ──closure──▶ [original hello function]
```

When you call `hello("bob")`, you are really calling `wrapper("bob")`, which calls the original inside.

### Call flow

```
you call hello("bob")
        │
        ▼
   wrapper("bob")
        │   ... code BEFORE ...
        ▼
   original hello("bob")   → "hello bob"
        │   ... code AFTER (.upper()) ...
        ▼
   returns "HELLO BOB"
```

---

## 5. When does the decorator run?

**At definition time, once.** The *decorator function itself* runs when Python executes the `def` statement. The *wrapper* runs on every call.

```python
def announce(fn):
    print(f"  decorating {fn.__name__}")          # runs ONCE, at definition
    def wrapper(*args, **kwargs):
        print(f"  calling {fn.__name__}")         # runs on EVERY call
        return fn(*args, **kwargs)
    return wrapper


print("before def")

@announce
def work():
    return 1

print("after def")

work()
work()
```

**Output**

```
before def
  decorating work
after def
  calling work
  calling work
```

`decorating work` appears between "before" and "after", proving the decorator ran the moment the function was defined, even though `work()` had not been called yet. This is why decorators are great for **registration** (section 11.4): they execute at import time.

---

## 6. The problem: a naive wrapper hides the original's identity

```python
def plain(fn):
    def wrapper(*args, **kwargs):
        "wrapper doc"
        return fn(*args, **kwargs)
    return wrapper


@plain
def add(a, b):
    "Add two numbers."
    return a + b


import inspect
print(add.__name__)             # wrapper          <- wrong!
print(add.__doc__)              # wrapper doc      <- wrong!
print(inspect.signature(add))   # (*args, **kwargs) <- wrong!
```

Because `add` now **is** `wrapper`, its name, docstring and signature are the wrapper's. This breaks debugging, `help()`, documentation tools, and frameworks that read signatures (FastAPI, pytest, and others).

### The fix: `functools.wraps`

```python
import functools, inspect

def good(fn):
    @functools.wraps(fn)                    # copies name, doc, module, etc. onto wrapper
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper


@good
def add2(a, b):
    "Add two numbers."
    return a + b

print(add2.__name__)              # add2
print(add2.__doc__)               # Add two numbers.
print(inspect.signature(add2))    # (a, b)
print(add2.__wrapped__.__name__)  # add2   <- the original is reachable here
```

> **Rule: always use `@functools.wraps(fn)` on your wrapper.** `wraps` is itself a decorator, which shows decorators are used to build decorators.

---

## 7. Decorators with arguments

Sometimes you want `@repeat(3)`. Look at what that really means:

```python
@repeat(3)
def hi(): ...

# is the same as:
hi = repeat(3)(hi)
#         └─┬─┘└┬┘
#     call #1   call #2
```

`repeat(3)` must **return a decorator**, which then receives the function. So you need **three levels** of functions:

```python
import functools

def repeat(times):                          # level 1: receives the decorator's ARGUMENTS
    def decorator(fn):                      # level 2: receives the FUNCTION
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):       # level 3: receives the CALL's arguments
            for _ in range(times):
                result = fn(*args, **kwargs)
            return result
        return wrapper
    return decorator


@repeat(3)
def hi():
    print("  hi")

hi()
```

**Output**

```
  hi
  hi
  hi
```

```
repeat(3)  ───▶  decorator  ───▶  wrapper
 (settings)       (the function)    (each call)
```

### A decorator that works with OR without parentheses

```python
import functools

def logged(_fn=None, *, prefix="LOG"):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            print(f"  [{prefix}] {fn.__name__}{args}")
            return fn(*args, **kwargs)
        return wrapper
    return decorator(_fn) if _fn else decorator


@logged                      # no parentheses
def one(x): return x

@logged(prefix="DEBUG")      # with arguments
def two(x): return x

one(1)    #   [LOG] one(1,)
two(2)    #   [DEBUG] two(2,)
```

---

## 8. Stacking multiple decorators

```python
def tag(name):
    def decorator(fn):
        def wrapper(*args, **kwargs):
            print(f"  enter {name}")
            result = fn(*args, **kwargs)
            print(f"  exit {name}")
            return result
        return wrapper
    return decorator


@tag("A")
@tag("B")
def core():
    print("  core")

core()
```

**Output**

```
  enter A
  enter B
  core
  exit B
  exit A
```

Decorators are applied **bottom-up** (closest to the function first), so this is:

```python
core = tag("A")(tag("B")(core))
```

```
core ──▶ [ wrapper A  [ wrapper B  [ original core ] ] ]
          outermost                  innermost
```

The outermost wrapper (A) runs first on the way in and last on the way out, like layers of an onion. **Order matters.** For example, `@login_required` above `@cache` behaves differently from the reverse.

---

## 9. Class-based decorators

Any object that is **callable** (has `__call__`) can act as a decorator. A class is handy when the decorator needs to **keep state**.

```python
import functools

class CountCalls:
    def __init__(self, fn):                 # runs once, at decoration time
        functools.update_wrapper(self, fn)  # same job as @wraps
        self.fn = fn
        self.count = 0

    def __call__(self, *args, **kwargs):    # runs on every call
        self.count += 1
        return self.fn(*args, **kwargs)


@CountCalls
def ping():
    return "pong"

ping()
ping()
print(ping.count)      # 2
print(ping.__name__)   # ping
```

Here `ping = CountCalls(ping)`, so `ping` is now an **instance** of `CountCalls`, and calling it runs `__call__`.

### Gotcha: a class-based decorator on a method

```python
class Greeter:
    @CountCalls
    def hi(self):
        return "hi"

Greeter().hi()
# TypeError: Greeter.hi() missing 1 required positional argument: 'self'
```

Why? Normal functions are **descriptors** (they have `__get__`), and that is how `self` gets filled in automatically (see [descriptor.md](descriptor.md)). A `CountCalls` instance has no `__get__`, so Python never binds `self`.

The fix is to give the decorator a `__get__`:

```python
import functools

class CountCalls2:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)
        self.fn = fn
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.fn(*args, **kwargs)

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return functools.partial(self.__call__, obj)    # bind `self` manually


class G2:
    @CountCalls2
    def hi(self):
        return "hi"

print(G2().hi())     # hi
```

**Function-based decorators avoid this problem automatically**, because the `wrapper` they return is a normal function, and normal functions are already descriptors. For that reason, prefer function-based decorators unless you need class-based state.

---

## 10. Decorating classes, and built-in decorators

A decorator can also receive a **class** and return it (modified):

```python
def add_repr(cls):
    cls.__repr__ = lambda self: f"{cls.__name__}({self.__dict__})"
    return cls


@add_repr
class P:
    def __init__(self, x):
        self.x = x

print(P(3))      # P({'x': 3})
```

Python's built-in decorators use the same mechanism:

| Decorator | What it does |
|---|---|
| `@staticmethod` | stops `self` from being passed |
| `@classmethod` | passes the class instead of the instance |
| `@property` | turns a method into an attribute (implemented with a descriptor) |
| `@functools.wraps` | copies metadata onto a wrapper |
| `@functools.lru_cache` | caches results |
| `@dataclasses.dataclass` | generates `__init__`, `__repr__`, and others |
| `@contextlib.contextmanager` | turns a generator into a `with` block |

---

## 11. Real-world examples

### 11.1 Timer

```python
import functools, time

def timer(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        print(f"  {fn.__name__} took {time.perf_counter() - start:.2f}s")
        return result
    return wrapper


@timer
def slow():
    time.sleep(0.2)
    return "done"

print(slow())
#   slow took 0.20s
# done
```

### 11.2 Retry on failure

```python
import functools

def retry(times=3, exceptions=(Exception,)):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as e:
                    print(f"  attempt {attempt} failed: {e}")
                    if attempt == times:
                        raise                   # give up on the last attempt
        return wrapper
    return decorator


calls = {"n": 0}

@retry(times=3)
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise ConnectionError("network down")
    return "ok"

print(flaky())
#   attempt 1 failed: network down
#   attempt 2 failed: network down
# ok
```

### 11.3 Permission check (like web frameworks do)

```python
import functools

def require_role(role):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(user, *args, **kwargs):
            if user.get("role") != role:
                raise PermissionError(f"{user['name']} is not {role}")
            return fn(user, *args, **kwargs)
        return wrapper
    return decorator


@require_role("admin")
def delete_all(user):
    return "deleted"

print(delete_all({"name": "root", "role": "admin"}))     # deleted

try:
    delete_all({"name": "bob", "role": "user"})
except PermissionError as e:
    print("denied:", e)                                  # denied: bob is not admin
```

### 11.4 Registration (the idea behind web routes)

This decorator does not wrap anything. It just records the function and returns it unchanged. It works because decorators run **at definition time**.

```python
ROUTES = {}

def route(path):
    def decorator(fn):
        ROUTES[path] = fn
        return fn                    # return the original, untouched
    return decorator


@route("/home")
def home():
    return "home page"

@route("/about")
def about():
    return "about page"

print(list(ROUTES))            # ['/home', '/about']
print(ROUTES["/home"]())       # home page
```

Flask's `@app.route(...)` and FastAPI's `@app.get(...)` work on this same principle.

### 11.5 Caching with `lru_cache`

```python
import functools

@functools.lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(80))             # 23416728348467685  (instant; without the cache this would take ages)
print(fib.cache_info())    # CacheInfo(hits=78, misses=81, maxsize=None, currsize=81)
```

Each distinct input is computed once and then served from memory.

### 11.6 Decorating `async` functions

A normal wrapper would call the coroutine without awaiting it. For `async def` functions, the wrapper must itself be `async` and use `await`:

```python
import asyncio, functools, time

def atimer(fn):
    @functools.wraps(fn)
    async def wrapper(*args, **kwargs):          # async wrapper
        start = time.perf_counter()
        result = await fn(*args, **kwargs)       # await the original
        print(f"  {fn.__name__} took {time.perf_counter() - start:.1f}s")
        return result
    return wrapper


@atimer
async def fetch():
    await asyncio.sleep(0.2)
    return "data"

print(asyncio.run(fetch()))
#   fetch took 0.2s
# data
```

---

## 12. Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Forgetting `functools.wraps` | name, docstring and signature are lost | add `@functools.wraps(fn)` |
| Calling instead of returning: `return wrapper()` | the decorated name becomes the *result*, not a function | `return wrapper` (no parentheses) |
| Forgetting to `return` the result inside `wrapper` | the decorated function always returns `None` | `return fn(*args, **kwargs)` |
| Writing `@repeat` when it needs arguments | the function is passed as `times` | write `@repeat(3)` |
| Class-based decorator on a method | `missing 1 required positional argument: 'self'` | add `__get__`, or use a function decorator |
| Decorating an `async def` with a sync wrapper | you get a coroutine that was never awaited | use an `async def wrapper` with `await` |
| Wrong stacking order | unexpected behaviour (e.g. cache before auth) | remember it applies **bottom-up** |
| Heavy work in the decorator body | slow imports (it runs at definition time) | do heavy work inside `wrapper` |

---

## 13. Quick summary

- A decorator is a function that **takes a function and returns a function**.
- `@deco` above `def f` is exactly `f = deco(f)`.
- It works thanks to three Python features: **first-class functions**, **closures** (the wrapper remembers `fn`), and `*args/**kwargs`.
- The decorator runs **once at definition time**. The **wrapper runs on every call**.
- Always use `@functools.wraps(fn)` to keep the original name, docstring and signature.
- A decorator **with arguments** needs three nested levels: `settings → function → call`.
- Stacked decorators apply **bottom-up** and run like layers of an onion.
- Classes can be decorators too (via `__call__`), but need `__get__` to work on methods.
- Decorators are used everywhere: timing, retry, caching, authentication, logging, and route registration.

## 14. Cheat sheet

```python
import functools

# 1. Basic template
def decorator(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        # ... before ...
        result = fn(*args, **kwargs)
        # ... after ...
        return result
    return wrapper

# 2. Template with arguments
def decorator_with_args(arg):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            return fn(*args, **kwargs)
        return wrapper
    return decorator

# 3. Class-based template
class Deco:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)
        self.fn = fn
    def __call__(self, *args, **kwargs):
        return self.fn(*args, **kwargs)
```

```
@a            f = a(b(c(f)))
@b      ==    applied bottom-up (c first), executed top-down (a's wrapper runs first)
@c
def f(): ...
```

## 15. Practice exercises

1. Write a `@debug` decorator that prints the function name, its arguments, and its return value on every call.
2. Write `@validate_positive` that raises `ValueError` if any numeric argument is `<= 0`.
3. Write `@rate_limit(calls=3)` that allows a function to run only 3 times, then raises an error. (Hint: keep a counter in the closure.)
4. Write `@memoize` yourself using a dictionary (do not use `lru_cache`). Test it on `fib(35)`, with and without the decorator, and compare the timing.
5. Predict the output before running: apply `@tag("A")`, `@tag("B")` and `@tag("C")` to one function. In what order do enter and exit print?
6. Write a single decorator that works on both normal functions and `async def` functions. (Hint: `inspect.iscoroutinefunction(fn)`.)
