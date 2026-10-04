# How Metaclasses Work Internally in Python (and When to Use Them)

> A beginner-friendly guide. Every code block below can be copied and run as-is.

---

## 1. What is a metaclass? (the one-line answer)

A **metaclass is the class of a class.** Just as a class builds instances, a metaclass builds classes.

```
metaclass  ──creates──▶  class  ──creates──▶  instance
  (type)                  (Dog)                (my_dog)
```

---

## 2. First idea: in Python, classes are objects too

Everything in Python is an object, and that includes classes. If a class is an object, then something must have created it. That something is `type`.

```python
class Dog:
    pass

d = Dog()

print(type(d))      # <class '__main__.Dog'>   -> d is an instance of Dog
print(type(Dog))    # <class 'type'>           -> Dog is an instance of type
print(type(type))   # <class 'type'>           -> type is an instance of itself

print(isinstance(Dog, type))   # True
print(isinstance(type, type))  # True
```

So `type` is the **default metaclass**. Whenever you write `class Something:`, Python uses `type` to build it.

---

## 3. You can create a class without the `class` keyword

`type` can be called with three arguments: `type(name, bases, attributes)`.

```python
def bark(self):
    return "Woof"

Dog2 = type("Dog2", (object,), {"legs": 4, "bark": bark})

print(Dog2().legs)      # 4
print(Dog2().bark())    # Woof
print(Dog2.__name__)    # Dog2
```

This is exactly what Python does behind the scenes when it sees a `class` statement. The `class` keyword is just friendly syntax over this call.

| Argument | Meaning |
|---|---|
| `name` | the class name (a string) |
| `bases` | a tuple of parent classes |
| `attributes` | a dict of everything in the class body (methods, variables) |

---

## 4. What happens internally when Python reads `class Foo:`

Python follows these steps, in order:

```
class Foo(Base, metaclass=Meta):
    x = 1
    def hello(self): ...
```

1. **Pick the metaclass.** Use `metaclass=` if given, otherwise the metaclass of the parent classes, otherwise `type`.
2. **`Meta.__prepare__(name, bases)`** returns an empty dict (the *namespace*) for the class body to fill.
3. **Run the class body.** The code inside `class Foo:` executes and fills the namespace with `x`, `hello`, etc.
4. **`Meta.__new__(mcs, name, bases, namespace)`** creates the class object.
5. **`Meta.__init__(cls, name, bases, namespace)`** initializes the class object.
6. The result is bound to the name `Foo`.

Later, when you write `Foo()`, Python calls **`Meta.__call__(Foo, ...)`**, which in turn runs `Foo.__new__` and `Foo.__init__` to build the instance.

### See it live

```python
class Spy(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kw):
        print(f"1. __prepare__ : name={name}")
        return {}

    def __new__(mcs, name, bases, namespace, **kw):
        keys = [k for k in namespace if not k.startswith("__")]
        print(f"3. __new__     : name={name}, keys={keys}")
        return super().__new__(mcs, name, bases, namespace)

    def __init__(cls, name, bases, namespace, **kw):
        print(f"4. __init__    : cls={cls.__name__}")
        super().__init__(name, bases, namespace)

    def __call__(cls, *args, **kw):
        print(f"5. __call__    : creating instance of {cls.__name__}")
        return super().__call__(*args, **kw)


class Demo(metaclass=Spy):
    print("2. class body running")
    x = 1
    def hello(self): pass

print("--- now creating an instance ---")
Demo()
```

**Output**

```
1. __prepare__ : name=Demo
2. class body running
3. __new__     : name=Demo, keys=['x', 'hello']
4. __init__    : cls=Demo
--- now creating an instance ---
5. __call__    : creating instance of Demo
```

Notice that steps 1 to 4 happen **when the class is defined**, not when you make an instance. That is the key insight about metaclasses: they run code at **class-creation time**.

### Which method does what?

| Method (on the metaclass) | Runs when | Typical use |
|---|---|---|
| `__prepare__` | before the class body runs | customize the namespace (rare) |
| `__new__` | the class is **created** | **modify or validate** the class before it exists |
| `__init__` | right after the class is created | tweak the finished class |
| `__call__` | you do `Foo()` (make an instance) | control instance creation, e.g. Singleton |

---

## 5. Real use case 1: enforce rules on subclasses

Goal: every subclass of `Animal` must define a `speak()` method, and we want the error at **class definition time**, not later at runtime.

```python
class RequireSpeak(type):
    def __new__(mcs, name, bases, ns):
        # `bases` is empty only for the root class (Animal itself), so skip it
        if bases and "speak" not in ns:
            raise TypeError(f"{name} must define a speak() method")
        return super().__new__(mcs, name, bases, ns)


class Animal(metaclass=RequireSpeak):
    pass


class Cat(Animal):
    def speak(self):
        return "Meow"

print(Cat().speak())        # Meow

try:
    class Fish(Animal):     # no speak() -> error right here
        pass
except TypeError as e:
    print("Error:", e)      # Error: Fish must define a speak() method
```

---

## 6. Real use case 2: auto-register plugins

Goal: every time someone defines a new plugin class, add it to a registry automatically, with no manual registration code.

```python
class PluginMeta(type):
    registry = {}

    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        if bases:                                   # skip the base class
            mcs.registry[name.lower()] = cls
        return cls


class Plugin(metaclass=PluginMeta):
    pass

class CsvExporter(Plugin):
    pass

class JsonExporter(Plugin):
    pass


print(PluginMeta.registry)
# {'csvexporter': <class '__main__.CsvExporter'>,
#  'jsonexporter': <class '__main__.JsonExporter'>}

exporter = PluginMeta.registry["csvexporter"]()     # look up by name and build
```

This pattern is common in frameworks (for example, ORMs registering models).

---

## 7. Real use case 3: Singleton (using `__call__`)

Goal: no matter how many times you call `Config()`, you always get the same object.

```python
class Singleton(type):
    _instances = {}

    def __call__(cls, *args, **kw):
        if cls not in Singleton._instances:
            Singleton._instances[cls] = super().__call__(*args, **kw)
        return Singleton._instances[cls]


class Config(metaclass=Singleton):
    def __init__(self):
        print("  Config created")


a = Config()          # prints "Config created"
b = Config()          # nothing printed, reuses the same object
print(a is b)         # True
```

`Config()` triggers `Singleton.__call__`, so we can decide whether to build a new instance or hand back the existing one.

---

## 8. Real use case 4: rewrite the class automatically

Goal: automatically turn every attribute name into UPPERCASE.

```python
class Upper(type):
    def __new__(mcs, name, bases, ns):
        new_ns = {k if k.startswith("__") else k.upper(): v
                  for k, v in ns.items()}
        return super().__new__(mcs, name, bases, new_ns)


class Const(metaclass=Upper):
    pi = 3.14


print(Const.PI)              # 3.14
print(hasattr(Const, "pi"))  # False
```

Because `__new__` receives the namespace **before** the class exists, it can change it freely.

---

## 9. Do you actually need a metaclass? Try `__init_subclass__` first

Since Python 3.6, many metaclass jobs have a simpler alternative. This registry does the same thing as section 6 without a metaclass:

```python
class Base:
    registry = []

    def __init_subclass__(cls, **kw):
        super().__init_subclass__(**kw)
        Base.registry.append(cls.__name__)

class A(Base): pass
class B(Base): pass

print(Base.registry)    # ['A', 'B']
```

**Rule of thumb:**

| You want to... | Use |
|---|---|
| Run code when a subclass is defined | `__init_subclass__` |
| Validate or register subclasses | `__init_subclass__` |
| Modify one class | a class decorator |
| Control the namespace, or `Foo()` itself (Singleton) | **metaclass** |
| Change how the class is built at a deep level | **metaclass** |

> *"Metaclasses are deeper magic than 99% of users should ever worry about."* (Tim Peters)

---

## 10. Gotcha: metaclass conflicts

A class can have only one metaclass. If you inherit from two classes with unrelated metaclasses, Python refuses:

```python
class M1(type): pass
class M2(type): pass

class X(metaclass=M1): pass
class Y(metaclass=M2): pass

try:
    class Z(X, Y): pass
except TypeError as e:
    print(e)
# metaclass conflict: the metaclass of a derived class must be a
# (non-strict) subclass of the metaclasses of all its bases
```

Fix: create a new metaclass that inherits from both `M1` and `M2`.

---

## 11. Quick summary

- Classes are objects, and `type` is the class that creates them (the default metaclass).
- `class Foo:` is shorthand for `type("Foo", bases, namespace)`.
- A custom metaclass subclasses `type` and hooks into class creation: `__prepare__`, then the class body, then `__new__`, then `__init__`.
- `__call__` on a metaclass controls what happens when you make an **instance**.
- Metaclass code runs at **class-definition time**.
- Use metaclasses for enforcing rules, auto-registration, Singleton, and rewriting classes. Prefer `__init_subclass__` or a class decorator when they are enough.

## 12. Cheat sheet

```python
class Meta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kw): ...    # supply the namespace dict
    def __new__(mcs, name, bases, ns, **kw): ...    # build the class (modify ns here)
    def __init__(cls, name, bases, ns, **kw): ...   # finish the class
    def __call__(cls, *args, **kw): ...             # runs on Foo() -> make an instance

class Foo(metaclass=Meta): ...
```

## 13. Practice exercises

1. Write a metaclass that raises an error if a class name does not start with a capital letter.
2. Write a metaclass that automatically adds a `describe()` method to every class it creates.
3. Write a metaclass that counts how many instances of each class have been created.
4. Predict the output: in the `Spy` example, what would be printed if `Demo` had a subclass `Child(Demo)`? (Hint: the metaclass is inherited.)
