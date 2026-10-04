# How Descriptors Work Internally in Python

> A beginner-friendly guide. Every code block below can be copied and run as-is.

---

## 1. What is a descriptor? (the one-line answer)

A **descriptor** is an object that **controls what happens when you read, write, or delete an attribute** of another object.

It does this by defining any of these three special methods:

| Method | Runs when you... | Example trigger |
|---|---|---|
| `__get__(self, obj, objtype)` | **read** the attribute | `print(d.x)` |
| `__set__(self, obj, value)` | **write** the attribute | `d.x = 10` |
| `__delete__(self, obj)` | **delete** the attribute | `del d.x` |

If an object defines at least one of these, it is a descriptor.

You have already used descriptors without knowing it. `@property`, `@classmethod`, `@staticmethod`, and even normal methods (`def`) are all built using descriptors.

---

## 2. The simplest possible descriptor

A descriptor only works when it is a **class attribute** of another class.

```python
class Verbose:
    def __get__(self, obj, objtype=None):
        print(f"  __get__ called  (obj={obj}, objtype={objtype.__name__})")
        return 42

    def __set__(self, obj, value):
        print(f"  __set__ called  (obj={obj}, value={value})")


class Demo:
    x = Verbose()            # <-- descriptor lives on the CLASS

    def __repr__(self):
        return "<Demo>"


d = Demo()

print("Reading d.x:")
print("  result:", d.x)

print("Writing d.x = 10:")
d.x = 10

print("Reading Demo.x (from the class, not an instance):")
Demo.x
```

**Output**

```
Reading d.x:
  __get__ called  (obj=<Demo>, objtype=Demo)
  result: 42
Writing d.x = 10:
  __set__ called  (obj=<Demo>, value=10)
Reading Demo.x (from the class, not an instance):
  __get__ called  (obj=None, objtype=Demo)
```

### What just happened?

- `d.x` did **not** simply look up a stored value. Python noticed `x` is a descriptor and called `Verbose.__get__` for us.
- `d.x = 10` did **not** store 10. Python called `Verbose.__set__` instead.
- When accessed from the class (`Demo.x`), `obj` is `None`. That is how a descriptor knows whether it was reached through an instance or the class.

### The parameters, explained

| Parameter | Meaning |
|---|---|
| `self` | the **descriptor** object itself (the `Verbose()` instance) |
| `obj` | the **instance** you accessed it through (`d`), or `None` if accessed via the class |
| `objtype` | the **class** (`Demo`) |
| `value` | the value being assigned (in `__set__`) |

> **Common beginner mix-up:** `self` is the descriptor, **not** the `Demo` instance. The `Demo` instance is `obj`.

---

## 3. What Python does internally (the lookup process)

When you write `d.x`, Python calls `type(d).__getattribute__(d, "x")`. Roughly, it follows these steps:

```
d.x
 │
 ├─ 1. Look up "x" on the CLASS (and its parents, via the MRO)
 │
 ├─ 2. Found, and it is a DATA descriptor (has __set__ or __delete__)?
 │       └─ YES → call its __get__ and STOP.            (highest priority)
 │
 ├─ 3. Is "x" in the INSTANCE's own __dict__ (d.__dict__)?
 │       └─ YES → return that value.
 │
 ├─ 4. Found on the class and it is a NON-DATA descriptor (only __get__)?
 │       └─ YES → call its __get__.
 │
 ├─ 5. Found on the class and it is a plain value?
 │       └─ YES → return it.
 │
 └─ 6. Nothing found → raise AttributeError
```

The key idea: **data descriptors beat the instance `__dict__`; non-data descriptors lose to it.**

Here is a pseudo-implementation of the same logic in plain Python:

```python
def my_getattr(obj, name):
    cls = type(obj)
    class_attr = None
    for klass in cls.__mro__:                     # search class + parents
        if name in klass.__dict__:
            class_attr = klass.__dict__[name]
            break

    has_get = class_attr is not None and hasattr(type(class_attr), "__get__")
    is_data = has_get and (hasattr(type(class_attr), "__set__")
                           or hasattr(type(class_attr), "__delete__"))

    if is_data:                                   # step 2
        return type(class_attr).__get__(class_attr, obj, cls)
    if name in obj.__dict__:                      # step 3
        return obj.__dict__[name]
    if has_get:                                   # step 4
        return type(class_attr).__get__(class_attr, obj, cls)
    if class_attr is not None:                    # step 5
        return class_attr
    raise AttributeError(name)                    # step 6
```

(The real version lives in C inside CPython, but the logic is the same.)

Writing (`d.x = 10`) is simpler: Python looks for a **data descriptor** named `x` on the class. If found, it calls `__set__`. If not, it stores the value in `d.__dict__`.

---

## 4. Data vs non-data descriptors

```python
class NonData:                       # only __get__
    def __get__(self, obj, objtype=None):
        return "from non-data descriptor"


class Data(NonData):                 # __get__ AND __set__
    def __set__(self, obj, value):
        pass


class A:
    nd = NonData()

class B:
    d = Data()


a = A()
print(a.nd)                          # from non-data descriptor
a.__dict__["nd"] = "instance value"
print(a.nd)                          # instance value   <-- instance wins!

b = B()
b.__dict__["d"] = "instance value"
print(b.d)                           # from non-data descriptor  <-- descriptor wins!
```

**Output**

```
from non-data descriptor
instance value
from non-data descriptor
```

| Type | Methods defined | Beats instance `__dict__`? |
|---|---|---|
| Non-data descriptor | only `__get__` | No, the instance wins |
| Data descriptor | `__set__` and/or `__delete__` | Yes, the descriptor wins |

---

## 5. A real-world example: validation

Problem: we want `price` and `quantity` to always be positive, and we do not want to repeat the same check for every attribute.

A descriptor solves this once, and we reuse it everywhere.

```python
class Positive:
    def __set_name__(self, owner, name):
        # Python calls this automatically when the class is created.
        # `name` is the attribute name ("price", "quantity", ...)
        self.private_name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:              # accessed from the class
            return self
        return getattr(obj, self.private_name)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"must be positive, got {value}")
        setattr(obj, self.private_name, value)


class Product:
    price = Positive()
    quantity = Positive()

    def __init__(self, name, price, quantity):
        self.name = name
        self.price = price           # triggers Positive.__set__
        self.quantity = quantity     # triggers Positive.__set__


p = Product("Pen", 10, 5)
print(p.price, p.quantity)           # 10 5
print(p.__dict__)                    # {'name': 'Pen', '_price': 10, '_quantity': 5}

try:
    p.price = -3
except ValueError as e:
    print("Error:", e)               # Error: must be positive, got -3

try:
    Product("Bad", 5, 0)
except ValueError as e:
    print("Error:", e)               # Error: must be positive, got 0
```

### Step-by-step: what happens on `p.price = -3`?

1. Python looks up `price` on `Product` and finds a `Positive()` object.
2. `Positive` defines `__set__`, so it is a **data descriptor**.
3. Python calls `Positive.__set__(descriptor, p, -3)`.
4. `-3 <= 0`, so it raises `ValueError`. Nothing is stored.

### Why store the value as `_price` and not `price`?

The real value must live somewhere. If `__set__` did `setattr(obj, "price", value)`, it would call `__set__` again, and again, forever (**infinite recursion**). So we store it under a different name (`_price`) in the instance `__dict__`.

### What is `__set_name__`?

It is a hook added in Python 3.6. When Python builds the `Product` class, it tells each descriptor the name it was assigned to. This saves you from writing `Positive("price")` and typing the name twice.

---

## 6. Where descriptors hide in Python

### 6.1 Normal methods are descriptors

Every function has a `__get__` method. That is how `self` gets filled in automatically.

```python
class T:
    def hi(self):
        pass

t = T()
print(type(T.__dict__["hi"]))               # <class 'function'>
print(hasattr(T.__dict__["hi"], "__get__")) # True

# What t.hi really does:
print(T.__dict__["hi"].__get__(t, T))
# <bound method T.hi of <__main__.T object at 0x...>>
```

`function.__get__(t, T)` returns a **bound method**, which is the function with `t` already attached as `self`. That is why you can write `t.hi()` without passing `t`.

### 6.2 `@property` is a descriptor

Here is a simplified version you can build yourself:

```python
class MyProperty:
    def __init__(self, fget):
        self.fget = fget

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return self.fget(obj)           # call the decorated function


class Circle:
    def __init__(self, r):
        self.r = r

    @MyProperty                         # same as: area = MyProperty(area)
    def area(self):
        return 3.14159 * self.r ** 2


print(Circle(2).area)                   # 12.56636  (no parentheses!)
```

### 6.3 A cached attribute (compute once, reuse)

This is a non-data descriptor, and it works because the instance `__dict__` wins on later lookups.

```python
class Cached:
    def __init__(self, func):
        self.func = func
        self.name = func.__name__

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        value = self.func(obj)
        obj.__dict__[self.name] = value  # store on the instance
        return value                     # next time, __dict__ is found first


class Report:
    @Cached
    def data(self):
        print("  computing...")
        return [1, 2, 3]


r = Report()
print(r.data)    # computing...  then [1, 2, 3]
print(r.data)    # [1, 2, 3]  (no recomputation)
```

The first access runs `__get__`, which saves the result in `r.__dict__["data"]`. On the second access, Python finds it in the instance `__dict__` first (step 3 of the lookup), so the descriptor is skipped.

This is essentially how `functools.cached_property` works.

---

## 7. Quick summary

- A descriptor is a class attribute whose class defines `__get__`, `__set__`, or `__delete__`.
- Python calls these methods automatically when you access the attribute through an instance.
- **Data descriptor** (`__set__` or `__delete__`) has priority over the instance `__dict__`. **Non-data descriptor** (only `__get__`) does not.
- Use descriptors to **reuse attribute logic** such as validation, caching, and type checks across many attributes or classes.
- `property`, `classmethod`, `staticmethod`, and methods are all descriptors.

## 8. Cheat sheet

```python
class Descriptor:
    def __set_name__(self, owner, name): ...   # learn the attribute name
    def __get__(self, obj, objtype=None): ...  # obj is None -> class access
    def __set__(self, obj, value): ...         # makes it a DATA descriptor
    def __delete__(self, obj): ...             # also makes it DATA
```

## 9. Practice exercises

1. Write a `TypedAttribute(int)` descriptor that raises `TypeError` if the wrong type is assigned.
2. Write a `ReadOnly` descriptor that raises `AttributeError` on any assignment.
3. Write a `Logged` descriptor that prints every time an attribute is read or written.
4. Predict the output before running: what happens if you add `__set__` to the `Cached` descriptor above? (Hint: re-read section 4.)
