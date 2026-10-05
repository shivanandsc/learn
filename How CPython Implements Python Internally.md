# How CPython Implements Python Internally

> A beginner-friendly guide. Every code block below can be copied and run as-is.
> All outputs were measured on **CPython 3.12.3, 64-bit Linux**. Opcode names and sizes can differ slightly between versions.
> Related reading: [GIL](gil.md), [memory management](memory-management.md), [descriptors](descriptor.md), [decorators](decorators.md), [event loop](event-loop-and-fastapi.md).

---

## 1. What is CPython? (the one-line answer)

**CPython is a program written in C that reads your Python code, translates it into a simple instruction format called *bytecode*, and then runs that bytecode on a small virtual machine.**

When you type `python script.py`, you are running a C program (the `python` executable) that does this work for you.

So is Python *compiled* or *interpreted*? **Both:**

```
 your .py source ──(compile)──▶ bytecode ──(interpret)──▶ result
                  step 1: fast, automatic     step 2: a big loop in C
```

---

## 2. "Python" vs "CPython"

**Python** is a *language specification* (rules about syntax and behaviour). **CPython** is one *implementation* of it, and it is the "reference implementation" that most people use.

| Implementation | Written in | Notes |
|---|---|---|
| **CPython** | C | the standard one from python.org |
| **PyPy** | RPython (a Python subset) | has a JIT compiler, often much faster on pure-Python code |
| **Jython** | Java | runs on the JVM |
| **IronPython** | C# | runs on .NET |
| **MicroPython** | C | tiny, for microcontrollers |
| **GraalPy** | Java | runs on GraalVM |

When people say "the GIL" or "reference counting", they mean **CPython behaviour**, not something every Python must do.

---

## 3. The whole pipeline at a glance

```
 source code      "total = price * 2 + 1"
      │
      ▼
 1. TOKENIZER      cuts text into tokens        NAME  OP  NAME  OP  NUMBER ...
      │
      ▼
 2. PARSER (PEG)   builds a tree (AST)          Assign( BinOp( ... ) )
      │
      ▼
 3. SYMBOL TABLE   decides: local? global?      which names live where
      │
      ▼
 4. COMPILER       walks the tree, emits        LOAD_NAME, BINARY_OP, STORE_NAME ...
                   bytecode, then optimizes it
      │
      ▼
 5. CODE OBJECT    bytecode + constants + names (cached in .pyc files)
      │
      ▼
 6. VIRTUAL MACHINE  the "eval loop" in C       executes one instruction at a time
```

Let's open each box with real code.

---

## 4. Step 1: Tokenizing

The tokenizer splits text into the smallest meaningful pieces.

```python
import tokenize, io

src = "total = price * 2 + 1\n"
for t in tokenize.generate_tokens(io.StringIO(src).readline):
    if t.type not in (tokenize.NEWLINE, tokenize.ENDMARKER):
        print(f"  {tokenize.tok_name[t.type]:8s} {t.string!r}")
```

```
  NAME     'total'
  OP       '='
  NAME     'price'
  OP       '*'
  NUMBER   '2'
  OP       '+'
  NUMBER   '1'
```

(The `tokenize` module is a pure-Python tool for you to explore. CPython's own tokenizer is written in C, but it produces the same kind of tokens.)

---

## 5. Step 2: Parsing into an AST

The parser turns tokens into a tree that captures the **structure and precedence** of the code. This tree is the **Abstract Syntax Tree (AST)**.

```python
import ast

tree = ast.parse("total = price * 2 + 1")
print(ast.dump(tree, indent=2))
```

```
Module(
  body=[
    Assign(
      targets=[
        Name(id='total', ctx=Store())],
      value=BinOp(
        left=BinOp(
          left=Name(id='price', ctx=Load()),
          op=Mult(),
          right=Constant(value=2)),
        op=Add(),
        right=Constant(value=1)))],
  type_ignores=[])
```

```
            Assign
            /    \
     total(Store)  BinOp(+)
                   /     \
              BinOp(*)    1
              /     \
           price     2
```

Notice that multiplication sits **deeper** in the tree than addition, which is exactly how operator precedence (`*` before `+`) is encoded.

Facts worth knowing:

- Since Python 3.9, CPython uses a **PEG parser** (PEP 617). The grammar is written in a file (`Grammar/python.gram`) and a tool **generates** the C parser from it.
- The AST node types are defined in `Parser/Python.asdl`.
- You can read, modify, and even compile ASTs from Python, which is how linters, formatters, and tools like `pytest`'s assert rewriting work.

---

## 6. Step 3: The symbol table

Before generating code, the compiler must know **where each name lives**. Is it a local variable, a global, or captured from an outer function?

```python
import symtable

code_src = """
x = 1
def f(a):
    b = a + x
    return b
"""
st = symtable.symtable(code_src, "<s>", "exec")
f = st.get_children()[0]               # the scope of function f
for s in f.get_symbols():
    print(f"  {s.get_name():2s} local={s.is_local()}  param={s.is_parameter()}  global={s.is_global()}")
```

```
  a  local=True  param=True  global=False
  b  local=True  param=False global=False
  x  local=False param=False global=True
```

This is why the bytecode for `a` and `b` (locals) uses fast array slots (`LOAD_FAST`), while `x` uses a slower dictionary lookup (`LOAD_GLOBAL`). It is also why assigning to a variable anywhere inside a function makes it local for the *whole* function (the classic `UnboundLocalError`).

---

## 7. Step 4: Compiling to bytecode

`compile()` runs the whole front half of the pipeline and returns a **code object**. The `dis` module shows its bytecode:

```python
import dis

co = compile("total = price * 2 + 1", "<demo>", "exec")
dis.dis(co)
print("constants:", co.co_consts)
print("names    :", co.co_names)
```

```
  1           2 LOAD_NAME                0 (price)
              4 LOAD_CONST               0 (2)
              6 BINARY_OP                5 (*)
             10 LOAD_CONST               1 (1)
             12 BINARY_OP                0 (+)
             16 STORE_NAME               1 (total)
             18 RETURN_CONST             2 (None)

constants: (2, 1, None)
names    : ('price', 'total')
```

(A leading `RESUME` instruction is also present.) Each line is one **instruction**: an opcode (what to do) plus an optional argument (usually an index into the constants or names tables).

### The compiler also optimizes

```python
def folded():
    return 24 * 60 * 60

import dis
dis.dis(folded)
print(folded.__code__.co_consts)
```

```
  2           2 RETURN_CONST             1 (86400)
(None, 86400)
```

The compiler did the multiplication **at compile time** (*constant folding*), so at runtime the function just returns `86400`.

### Inside the compiler (where to look in the source)

In recent CPython versions, the compile stage is split into several C files:

| File | Job |
|---|---|
| `Python/symtable.c` | build the symbol table |
| `Python/codegen.c` | walk the AST and emit instructions |
| `Python/flowgraph.c` | optimize the control-flow graph (dead code, jumps, folding) |
| `Python/assemble.c` | produce the final bytecode bytes and line tables |
| `Python/compile.c` | coordinates all of it |

---

## 8. Step 5: The code object

A **code object** is the compiled, read-only package for a block of code. Every function has one at `func.__code__`.

```python
def add(a, b):
    c = a + b
    return c

c = add.__code__
print(c.co_name)         # add
print(c.co_argcount)     # 2
print(c.co_varnames)     # ('a', 'b', 'c')    local variable names
print(c.co_consts)       # (None,)            constants used
print(c.co_stacksize)    # 2                  max depth of the value stack
print(c.co_code[:8])     # b'\x97\x00|\x00|\x01z\x00'   the raw bytecode bytes
```

| Field | Meaning |
|---|---|
| `co_code` | the raw bytecode (2 bytes per instruction on 3.12, plus inline cache entries) |
| `co_consts` | constants (numbers, strings, nested code objects) |
| `co_names` | global/attribute names used |
| `co_varnames` | local variable names |
| `co_stacksize` | how deep the value stack can get (the compiler works it out ahead of time) |
| `co_freevars` / `co_cellvars` | variables shared through closures (see [decorators.md](decorators.md)) |

**Function vs code object:** the code object is the *recipe*. The function object (created when `def` runs) wraps the recipe with a name, defaults, a reference to globals, and closure cells. One code object can back many function objects.

### `eval` and `exec` just run code objects

```python
print(eval(compile("2 + 3 * 4", "<e>", "eval")))      # 14

ns = {}
exec(compile("y = 10\nz = y * 2", "<x>", "exec"), ns)
print(ns["z"])                                         # 20
```

---

## 9. Step 6: The virtual machine (the "eval loop")

CPython's VM is a **stack machine**: instructions push values onto a stack, and other instructions pop them off and compute.

```python
import dis

def calc(a, b, c):
    return (a + b) * c

dis.dis(calc)
```

```
  LOAD_FAST    0 (a)
  LOAD_FAST    1 (b)
  BINARY_OP    0 (+)
  LOAD_FAST    2 (c)
  BINARY_OP    5 (*)
  RETURN_VALUE
```

Trace it with `calc(2, 3, 4)`:

| Instruction | Value stack afterwards |
|---|---|
| `LOAD_FAST a` | `[2]` |
| `LOAD_FAST b` | `[2, 3]` |
| `BINARY_OP +` | `[5]` (pops 2 and 3, pushes 5) |
| `LOAD_FAST c` | `[5, 4]` |
| `BINARY_OP *` | `[20]` |
| `RETURN_VALUE` | returns `20` |

### The loop itself

At the heart of CPython is one huge C function, `_PyEval_EvalFrameDefault` in `Python/ceval.c`. Conceptually it is just:

```
 for each frame:
     while True:
         instruction = bytecode[next_index]
         jump to the C code that implements that opcode
         (do the work: push, pop, call, jump...)
```

- The C code for each opcode is **not written by hand in `ceval.c`**. It is written in a small description language in `Python/bytecodes.c`, and a tool **generates** `Python/generated_cases.c.h` from it.
- On GCC and Clang, dispatch uses **computed gotos** (jumping straight to the next opcode's code) instead of a big `switch`, which is faster.
- Optionally, 3.14 can be built with a **tail-calling interpreter**, where each opcode is a separate small C function that tail-calls the next (needs a recent Clang, and is off by default on most builds).

### Build a tiny stack VM yourself

This mini VM has the same shape as the real thing:

```python
def run(code, consts, names, env):
    stack, pc = [], 0
    while pc < len(code):
        op, arg = code[pc]
        pc += 1
        if   op == "LOAD_NAME":  stack.append(env[names[arg]])
        elif op == "STORE_NAME": env[names[arg]] = stack.pop()
        elif op == "BINARY_ADD": b, a = stack.pop(), stack.pop(); stack.append(a + b)
        elif op == "BINARY_MUL": b, a = stack.pop(), stack.pop(); stack.append(a * b)
        elif op == "PRINT":      print(stack.pop())
        else: raise ValueError(op)
        print(f"  {op:12s} {str(arg):4s} stack={stack}")
    return env

# result = (a + b) * c
prog = [("LOAD_NAME", 0), ("LOAD_NAME", 1), ("BINARY_ADD", None),
        ("LOAD_NAME", 2), ("BINARY_MUL", None),
        ("STORE_NAME", 3), ("LOAD_NAME", 3), ("PRINT", None)]

run(prog, [], ["a", "b", "c", "result"], {"a": 2, "b": 3, "c": 4})
```

```
  LOAD_NAME    0    stack=[2]
  LOAD_NAME    1    stack=[2, 3]
  BINARY_ADD   None stack=[5]
  LOAD_NAME    2    stack=[5, 4]
  BINARY_MUL   None stack=[20]
  STORE_NAME   3    stack=[]
  LOAD_NAME    3    stack=[20]
20
  PRINT        None stack=[]
```

The real VM adds hundreds of opcodes, exception handling, calls, and a lot of speed tricks, but the idea is the same: **fetch an instruction, do it, move on**.

---

## 10. Frames and the call stack

Each function call creates a **frame**: a record holding the function's local variables, its value stack, and "where am I in the bytecode".

```python
import sys

def inner():
    f = sys._getframe()
    chain = []
    while f:
        chain.append(f.f_code.co_name)
        f = f.f_back                 # the caller's frame
    return chain

def middle(): return inner()
def outer():  return middle()

print(outer())      # ['inner', 'middle', 'outer', '<module>']
```

```
 <module> frame ──▶ outer frame ──▶ middle frame ──▶ inner frame   (top = running now)
```

- A traceback is just this chain printed out.
- Python limits the depth to prevent a runaway recursion from crashing the C stack:

```python
import sys
print(sys.getrecursionlimit())       # 1000

def rec(n): return rec(n + 1)
try:
    rec(0)
except RecursionError as e:
    print("RecursionError:", e)      # maximum recursion depth exceeded
```

- **Generators** keep their frame alive between `yield`s, which is how they pause and resume:

```python
def gen():
    yield 1
    yield 2

g = gen()
print(type(g), g.gi_frame is not None)   # <class 'generator'> True
print(next(g), g.gi_frame.f_lineno)      # 1 61   (paused at the first yield)
```

`async` functions (coroutines) work the same way, which is what the [event loop](event-loop-and-fastapi.md) relies on.

---

## 11. Everything is a `PyObject`

In C, every Python value, from `5` to a function to a class, is a C struct that **begins with the same two fields**:

```
 ┌─────────────────────────────┐
 │ reference count  (ob_refcnt)│   how many references point here
 ├─────────────────────────────┤
 │ type pointer     (ob_type)  │   which type object describes me
 ├─────────────────────────────┤
 │ ...type-specific data...    │   the digits of an int, the items of a list, ...
 └─────────────────────────────┘
```

Because the layout starts the same way, C code can hold a plain `PyObject *` pointer to anything and ask "what are you?" through `ob_type`. That is how a language with dynamic types is built in C. (Newer versions pack extra flag bits next to the count, and free-threaded builds use a different header. The idea is the same.)

You can see the real thing from Python with `ctypes`:

```python
import ctypes, sys

x = [1, 2, 3]
refcnt = ctypes.c_ssize_t.from_address(id(x)).value             # first field
type_ptr = ctypes.c_void_p.from_address(id(x) + 8).value        # second field

print("refcnt in memory :", refcnt)                  # 1
print("getrefcount      :", sys.getrefcount(x))      # 2   (+1 for the call's temporary)
print("type pointer is list :", type_ptr == id(list))   # True
print("id(x) is the address :", hex(id(x)))
```

- `id(x)` in CPython **is the memory address** of the object.
- The second field really does point at the `list` type object.

> Reading raw memory with `ctypes` is only a demo. Do not do this in real code.

---

## 12. Type objects and slots: how `a + b` finds its code

Each type (`int`, `list`, your own class) is itself an object, a C struct called `PyTypeObject`. It holds a table of **slots**, pointers to C functions for the standard operations:

| You write | CPython slot used | Python-level dunder |
|---|---|---|
| `a + b` | `nb_add` | `__add__` |
| `len(x)` | `sq_length` / `mp_length` | `__len__` |
| `x[i]` | `mp_subscript` | `__getitem__` |
| `x()` | `tp_call` | `__call__` |
| `x.name` | `tp_getattro` | `__getattribute__` |
| `repr(x)` | `tp_repr` | `__repr__` |
| `x == y` | `tp_richcompare` | `__eq__` etc. |
| `iter(x)` | `tp_iter` | `__iter__` |

So `BINARY_OP +` does not know about ints or strings. It asks the **type** of the left operand: "do you have an `nb_add`?"

```python
print(1 + 2, int.__add__(1, 2), (1).__add__(2))    # 3 3 3  (all identical)
print(type(int.__add__))                           # <class 'wrapper_descriptor'>
```

For built-in types, the slot points straight to a C function, and Python exposes it as the dunder method (`int.__add__` is a thin wrapper around it). For your own classes, CPython fills the slot with a small C function that **looks up and calls your Python `__add__`**:

```python
class Money:
    def __init__(self, v): self.v = v
    def __add__(self, o):  return Money(self.v + o.v)
    def __repr__(self):    return f"Money({self.v})"

print(Money(1) + Money(2))        # Money(3)
```

This is the same mechanism as [descriptors](descriptor.md) and [metaclasses](metaclass.md): Python's flexible behaviour is built from tables of function pointers and well-defined lookup rules.

---

## 13. How the core data types look inside

All implemented in C (see `Objects/`):

| Type | Internal design | Evidence you can measure |
|---|---|---|
| **int** | arbitrary-precision: an array of 30-bit "digits" | 28 bytes up to 30 bits, then +4 bytes for each extra 30 bits |
| **float** | a C `double` in a small object | 24 bytes |
| **str** | compact Unicode: 1, 2 or 4 bytes per character, chosen per string | see below |
| **list** | a dynamic **array of pointers** to objects, over-allocated | grows in steps (see [memory-management.md](memory-management.md)) |
| **tuple** | fixed array of pointers | smaller than the equivalent list |
| **dict** | a compact **hash table**, keeping insertion order | grows in steps |

```python
import sys

# ints: 30 bits per digit
for n in [0, 2**30 - 1, 2**30, 2**60, 2**90]:
    print(f"int {n.bit_length():3d} bits -> {sys.getsizeof(n)} bytes")
# int   0 bits -> 28 bytes
# int  30 bits -> 28 bytes
# int  31 bits -> 32 bytes
# int  61 bits -> 36 bytes
# int  91 bits -> 40 bytes

# strings: storage width depends on the widest character in the string
for ch, name in [("a", "ASCII"), ("é", "Latin-1"), ("日", "BMP"), ("😀", "astral")]:
    print(f"{name:8s} x100 -> {sys.getsizeof(ch * 100)} bytes")
# ASCII    x100 -> 141 bytes   (1 byte per char)
# Latin-1  x100 -> 157 bytes   (1 byte per char, bigger header)
# BMP      x100 -> 258 bytes   (2 bytes per char)
# astral   x100 -> 460 bytes   (4 bytes per char)

# tuple vs list
print(sys.getsizeof((1, 2, 3)), sys.getsizeof([1, 2, 3]))     # 64 88
```

Because `ints` are arbitrary-precision, `2**1000` just works. There is no overflow, but each operation is slower than a raw C integer.

### Why a built-in loop beats your own loop

`sum(range(100000))` runs its loop **inside C**. A Python `for` loop runs the eval loop (fetch, dispatch, push, pop) for every iteration:

```python
import timeit
print(timeit.timeit("sum(range(100000))", number=100))                       # ~0.12 s
print(timeit.timeit("t=0\nfor i in range(100000): t+=i", number=100))        # ~0.31 s
```

That is roughly **2.7x** slower on my machine for the same result. The rule "push work down into built-ins or C libraries like NumPy" comes straight from this design.

---

## 14. How CPython got faster: the adaptive interpreter

Early CPython executed every instruction generically. For `a + b` it had to check the types every single time. Since **Python 3.11** (PEP 659), the interpreter is **adaptive**:

1. A function starts out with generic instructions.
2. After it has run a few times ("warm"), the interpreter notices the types it keeps seeing.
3. It **rewrites the instruction in place** into a specialized fast version.
4. If the assumption breaks (say, a string suddenly arrives), it falls back safely.

You can watch it happen:

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)                                     # generic
#   BINARY_OP    0 (+)

for _ in range(1000):                            # warm it up with ints
    add(1, 2)

dis.dis(add, adaptive=True)                      # specialized
#   LOAD_FAST__LOAD_FAST   0 (a)
#   LOAD_FAST              1 (b)
#   BINARY_OP_ADD_INT      0 (+)                 <- specialized for ints!
```

Other specializations include `LOAD_ATTR_INSTANCE_VALUE` (fast attribute reads), `LOAD_GLOBAL_MODULE` (cached global lookups), and `CALL_PY_EXACT_ARGS` (fast calls of Python functions). The source for these lives in `Python/specialize.c`, with the fast versions defined in `Python/bytecodes.c`.

> **Practical lesson:** code that is **type-stable** (a function that always sees ints, say) runs faster than code that keeps changing types.

### The performance roadmap (version by version)

| Version | What changed |
|---|---|
| **3.11** | adaptive/specializing interpreter, cheaper frames, "zero-cost" exceptions |
| **3.12** | immortal objects (see [memory-management.md](memory-management.md)), per-interpreter GIL groundwork |
| **3.13** | optional **experimental JIT** (build flag, off by default); experimental free-threaded build |
| **3.14** | free-threaded build **officially supported**; tail-calling interpreter available as a build option (newer Clang); official macOS and Windows binaries include the **experimental JIT**, off by default (enable with `PYTHON_JIT=1`) |
| **3.15** | JIT upgraded (the release notes report roughly a 7-8% geometric-mean gain over the standard interpreter on x86-64 Linux); official Windows 64-bit binaries use the tail-calling interpreter. At the time of writing, 3.15 was in its release-candidate stage |

The JIT in CPython is a **small, "copy-and-patch" style JIT** that works on hot sequences of micro-operations, rather than a big rewrite like PyPy's. It is still marked experimental, and this area changes quickly, so always check the official "What's New" page for your version.

---

## 15. Imports and `.pyc` files: caching the compile step

Compiling takes time, so CPython **saves the code object** to disk the first time a module is imported, in a `__pycache__` folder.

```python
import py_compile, marshal, importlib.util

open("mod_demo.py", "w").write("X = 41\ndef f(): return X + 1\n")
py_compile.compile("mod_demo.py", cfile="mod_demo.pyc")

data = open("mod_demo.pyc", "rb").read()
print("magic:", data[:4])                                       # b'\xcb\r\r\n'
print("flags:", data[4:8])                                      # b'\x00\x00\x00\x00'
print("matches this interpreter:", data[:4] == importlib.util.MAGIC_NUMBER)   # True

code = marshal.loads(data[16:])           # after the 16-byte header: the code object
print(type(code), code.co_names)          # <class 'code'> ('X', 'f')

ns = {}
exec(code, ns)
print(ns["f"]())                          # 42

print(importlib.util.cache_from_source("pkg/mod.py"))
# pkg/__pycache__/mod.cpython-312.pyc
```

A `.pyc` file is:

```
 ┌───────────── 16-byte header ─────────────┐┌──── marshal data ────┐
 │ magic number │ flags │ mtime+size or hash ││ serialized code object│
 └───────────────────────────────────────────┘└──────────────────────┘
```

- The **magic number** identifies the bytecode version. A `.pyc` from another Python version is ignored and rebuilt.
- By default the header stores the source file's **modification time and size**. If the source changed, the `.pyc` is regenerated.
- The file name includes the implementation and version (`cpython-312`), so several versions can share one folder.
- **Important:** the main script you run (`python script.py`) is compiled in memory every time and is **not** cached. Only *imported* modules get `.pyc` files. Caching makes **startup** faster, not execution.
- `.pyc` is **not** machine code. It is bytecode that still needs the VM.

---

## 16. Extending CPython with C

Because CPython is C, you can write **extension modules** in C that Python imports like any other. This is how NumPy, parts of the standard library (`math`, `_json`, `_sqlite3`), and many others are so fast.

Here is a complete tiny extension (I compiled and ran this one):

```c
// spam.c
#include <Python.h>

static PyObject* add_ints(PyObject* self, PyObject* args) {
    long a, b;
    if (!PyArg_ParseTuple(args, "ll", &a, &b))   // parse two C longs from the arguments
        return NULL;                             // NULL means "an exception is set"
    return PyLong_FromLong(a + b);               // wrap the C result in a Python int
}

static PyMethodDef Methods[] = {
    {"add_ints", add_ints, METH_VARARGS, "Add two integers in C."},
    {NULL, NULL, 0, NULL}
};

static struct PyModuleDef mod = {PyModuleDef_HEAD_INIT, "spam", NULL, -1, Methods};

PyMODINIT_FUNC PyInit_spam(void) { return PyModule_Create(&mod); }
```

Build and use it:

```bash
gcc -shared -fPIC -O2 -I/usr/include/python3.12 spam.c -o spam$(python3-config --extension-suffix)
```

```python
import spam
print(spam.add_ints(2, 3))             # 5
print(spam.add_ints.__doc__)           # Add two integers in C.
print(type(spam.add_ints))             # <class 'builtin_function_or_method'>

try:
    spam.add_ints("a", 1)
except TypeError as e:
    print("TypeError:", e)             # 'str' object cannot be interpreted as an integer
```

What you see here is the whole contract:

- Every Python value in C is a `PyObject *`.
- Functions receive arguments as a tuple, and return a new `PyObject *` (or `NULL` to signal an exception).
- **You manage reference counts** in C when you hold onto objects (`Py_INCREF` / `Py_DECREF`), which is the main source of bugs in extensions. Tools like **Cython**, **pybind11**, **PyO3 (Rust)** and `cffi` exist to hide this work.
- While the C code runs, it holds the GIL unless it explicitly releases it. That is how NumPy can run big computations on several threads (see [gil.md](gil.md)).

---

## 17. A map of the CPython source code

You can read all of this yourself at https://github.com/python/cpython. I cloned the repo to check this layout (the `main` branch on this date is the in-development 3.16).

| Folder | What lives there |
|---|---|
| `Parser/` | tokenizer and the generated PEG parser (`Python.asdl` defines AST nodes) |
| `Python/` | the compiler, **the eval loop** (`ceval.c`, `bytecodes.c`), imports, GC, specialization, JIT support |
| `Objects/` | the built-in types: `longobject.c`, `listobject.c`, `dictobject.c`, `unicodeobject.c`, `typeobject.c`, `obmalloc.c` (the allocator) |
| `Include/` | C headers, including the public C API |
| `Modules/` | C modules of the standard library (`math`, `_json`, `itertools`, ...) |
| `Lib/` | the standard library written in Python (`os`, `json`, `asyncio`, ...) |
| `Grammar/` | the PEG grammar definition |
| `Tools/` | code generators and build helpers |

A few approximate file sizes on that checkout, to give a feel for the scale: `Objects/typeobject.c` about 13,000 lines, `Objects/dictobject.c` about 8,600 lines, and `Python/bytecodes.c` (the opcode definitions) about 6,600 lines.

**Where to start reading as a beginner:**

1. `Include/object.h` (the `PyObject` header)
2. `Objects/listobject.c` (a simple type)
3. `Python/bytecodes.c` (short, readable opcode definitions like `LOAD_FAST`)
4. `Python/ceval.c` (how the loop is wired together)

You can also see how many opcodes your version has:

```python
import dis
print(len(dis.opmap))      # 140 on 3.12
```

---

## 18. Putting it all together: life of one line

For `total = price * 2 + 1`:

```
 1. Tokenizer     NAME '=' NAME '*' NUMBER '+' NUMBER
 2. Parser        Assign(total, BinOp(BinOp(price * 2) + 1))
 3. Symtable      'price' and 'total' are module-level names
 4. Compiler      LOAD_NAME price; LOAD_CONST 2; BINARY_OP *; LOAD_CONST 1; BINARY_OP +; STORE_NAME total
 5. Code object   stored (and cached in .pyc when imported)
 6. VM            loops over the instructions:
                    - looks up 'price' → pushes the int object
                    - BINARY_OP * → asks type(int) for nb_multiply → C function → new int object
                    - ...
                    - STORE_NAME → binds the name 'total' to the result (refcount +1)
 7. Memory        reference counts adjust; the old objects are freed when counts reach 0
```

---

## 19. Quick summary

- **CPython** is the standard Python **implementation, written in C**. "Python" is the language; CPython is one way to run it.
- Code goes through: **tokenizer → PEG parser → AST → symbol table → compiler → bytecode (code object) → virtual machine**.
- The VM is a **stack machine** whose heart is the eval loop in `Python/ceval.c`, with opcode behaviour defined in `Python/bytecodes.c`.
- Each call makes a **frame**. Generators and coroutines keep their frames alive between pauses.
- Every value is a C `PyObject` with a **reference count** and a **type pointer**. Types hold **slots** (function pointers) that implement `+`, `len()`, `[]`, `()` and more.
- Since 3.11 the interpreter **specializes** hot bytecode, and newer versions add an experimental JIT and an optional tail-calling interpreter.
- Imported modules are cached as **`.pyc`** files (bytecode, not machine code). They speed up startup, not execution.
- C extensions plug directly into this design, which is why **NumPy and friends are fast**.
- Use `dis`, `ast`, `tokenize`, `symtable`, `compile`, `sys._getframe`, and `ctypes` to explore all of the above yourself.

## 20. Cheat sheet

```python
import tokenize, ast, symtable, dis, sys, marshal, py_compile, importlib.util

ast.parse(src); ast.dump(tree, indent=2)    # source -> AST
symtable.symtable(src, "<s>", "exec")        # scopes of names
co = compile(src, "<f>", "exec")             # source -> code object
dis.dis(func_or_code)                        # show bytecode
dis.dis(func, adaptive=True)                 # show specialized bytecode (after warm-up)
func.__code__                                # the code object of a function
co.co_consts; co.co_names; co.co_varnames    # tables the bytecode refers to
exec(co, namespace); eval(co)                # run a code object
sys._getframe(); frame.f_back; frame.f_code  # the call stack
sys.getrecursionlimit()                      # max frame depth
py_compile.compile("m.py")                   # write a .pyc
importlib.util.MAGIC_NUMBER                  # bytecode version tag
```

```
Pipeline:   source → tokens → AST → symtable → bytecode → [ .pyc cache ] → eval loop
Objects:    [ refcount | type pointer | data ]    (every value, always)
Operators:  a + b  →  BINARY_OP  →  type(a)->nb_add  →  C function (or your __add__)
```

## 21. Practice exercises

1. Run `dis.dis` on a function with an `if/else`, a `for` loop, and a `try/except`. Find the jump instructions and explain what each one does.
2. Use `ast.parse` on `1 + 2 * 3` and `(1 + 2) * 3`. How does the tree differ?
3. Write a function containing `x = 60 * 60` and check `co_consts` to prove constant folding happened.
4. Warm up a function that adds two ints, check `dis.dis(f, adaptive=True)`, then call it with two strings a thousand times and look again. What changed?
5. Extend the mini stack VM from section 9 with a `LOAD_CONST` opcode and a `JUMP_IF_FALSE` opcode, then write a program that prints a number only if a condition is true.
6. Import a module you wrote, find its `.pyc` in `__pycache__`, read the first 16 bytes, and confirm the magic number equals `importlib.util.MAGIC_NUMBER`.
7. Compile the C extension from section 16 and add a second function `multiply(a, b)`. What happens if you pass floats instead of ints?
8. Clone https://github.com/python/cpython (a shallow `--depth 1` clone is enough) and find the definition of `LOAD_FAST` in `Python/bytecodes.c`. How many lines is it?
