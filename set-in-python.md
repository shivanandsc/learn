# Set in Python

## 1. TL;DR
A `set` in Python is an **unordered, mutable collection of unique (hashable) elements**. It exists to efficiently test membership, remove duplicates, and perform mathematical set operations like union/intersection. Use it when you need **fast `in` checks** and uniqueness, not when you need ordering or indexing.

## 2. Background & Motivation
Many problems naturally involve **uniqueness**: deduplicating IDs, tracking “already visited” nodes, checking if an item is allowed/blocked, etc. Before sets, developers often used:

- **Lists**: easy to append, but membership checks are $O(n)$.
- **Dictionaries**: can emulate a set by storing keys with dummy values.

Python provides `set` (and `frozenset`) to make uniqueness and set algebra first-class, with typical membership operations averaging **$O(1)$** due to hashing.

## 3. Core Concepts

| Term | Meaning |
|---|---|
| Hashable | An object with a stable hash value and equality semantics (e.g., `int`, `str`, `tuple` of hashables). Required for set elements. |
| Unordered | A set has no stable positional order you can rely on (even though iteration order may appear consistent within a run). |
| Unique | Duplicate insertions are ignored; each element appears at most once. |
| Mutable | You can add/remove elements after creation (`set` is mutable). |
| `frozenset` | Immutable version of a set; can itself be used as a set element or dict key. |
| Membership test | Checking `x in s` efficiently via hashing. |

From first principles:
- A set answers: “Is this element present?” and “What elements are in A but not in B?”
- Internally, it’s built on a **hash table** (similar idea to `dict`), mapping hashed elements to slots.

Analogy (hardest idea: hashing + uniqueness):
- Think of a set like a library’s **card catalog** where each book title maps to a slot based on a computed code (hash). If the title already exists in that slot chain, it won’t be added again.

## 4. Theory (Deep Dive)

### 4.1 Internal Model (Hash Table)
Python sets are implemented with an **open-addressing hash table**.

- Each element has a hash: `h = hash(x)`
- The set stores elements in an array of slots.
- Collisions are handled by probing other slots.

Typical operations:
- **Lookup (`x in s`)**: compute hash, probe until found/empty.
- **Insert (`s.add(x)`)**: lookup; if not found, place into an empty slot.
- **Delete (`s.remove(x)`)**: locate; mark slot as deleted (tombstone) or restructure as needed.

### 4.2 Data Flow (Conceptual)
```mermaid
flowchart LR
  A[Element x] --> B[hash(x)]
  B --> C[Probe table index]
  C --> D{Slot empty?}
  D -- yes --> E[Insert x]
  D -- no --> F{Equal to existing?}
  F -- yes --> G[Already present (no-op)]
  F -- no --> C
```

### 4.3 Complexity & Performance
Let $n$ be number of elements.

| Operation | Average | Worst-case | Notes |
|---|---:|---:|---|
| `x in s` | $O(1)$ | $O(n)$ | Worst-case with many collisions; mitigated by good hashing. |
| `s.add(x)` | $O(1)$ | $O(n)$ | Occasional resize/rehash can be $O(n)$. |
| `s.remove(x)` | $O(1)$ | $O(n)$ | Use `discard` to avoid exceptions. |
| Union / Intersection | $O(n + m)$ | $O(nm)$ | Practically near-linear; depends on hashing and sizes. |

Space: sets trade memory for speed; hash tables require unused capacity to keep probing efficient.

### 4.4 Invariants & Guarantees
- **Uniqueness invariant**: at most one equal element exists.
- Elements must be **hashable** and should be **effectively immutable** while inside a set (don’t mutate objects in a way that changes `__hash__`/`__eq__`).
- Ordering is not part of the API contract.

Trade-offs:
- Pros: fast membership, deduplication, set algebra.
- Cons: not indexable, higher memory overhead, requires hashable elements.

## 5. Practical Usage

### 5.1 Minimal “Hello World”
```python
s = set()

s.add("apple")
s.add("apple")  # duplicate ignored

print(s)               # {'apple'}
print("apple" in s)    # True
print("banana" in s)   # False
```

### 5.2 Realistic Example: De-duplicate while preserving original order
A set is great for membership, but it doesn’t preserve insertion order as a guaranteed contract. To dedupe while preserving first-seen order, use a set + list:

```python
def dedupe_preserve_order(items):
    seen = set()
    out = []
    for x in items:
        if x not in seen:
            seen.add(x)
            out.append(x)
    return out

print(dedupe_preserve_order([3, 1, 3, 2, 1]))  # [3, 1, 2]
```

### 5.3 Typical Operations
```python
a = {1, 2, 3}
b = {3, 4}

print(a | b)   # union: {1,2,3,4}
print(a & b)   # intersection: {3}
print(a - b)   # difference: {1,2}
print(a ^ b)   # symmetric difference: {1,2,4}

# subset/superset
print({1, 2} <= a)  # True
print(a >= {2})     # True
```

### 5.4 `remove` vs `discard`
```python
s = {1, 2}

s.discard(3)  # no error
# s.remove(3) # KeyError
```

### 5.5 `frozenset` for immutability / set of sets
```python
groups = {
    frozenset({"read", "write"}),
    frozenset({"read"})
}

print(frozenset({"read"}) in groups)  # True
```

## 6. Common Pitfalls & Best Practices

| Mistake | Why it happens | Fix |
|---|---|---|
| Expecting stable ordering | Sets are unordered by contract | Use `list`, `tuple`, or `dict` (Py3.7+ preserves insertion order for dicts). |
| Using unhashable elements (e.g., lists, dicts) | They are mutable and not hashable | Convert to `tuple` or use `frozenset` for nested sets. |
| Mutating an object after adding it | Hash/equality may change, “losing” the item | Only store immutable/hash-stable objects. |
| Using `remove` when absence is normal | `remove` raises `KeyError` | Use `discard` or `if x in s:` guard. |
| Confusing `{}` with empty set | `{}` is an empty dict | Use `set()` for empty set. |

## 7. Comparison with Alternatives

| Need | Use | Why |
|---|---|---|
| Fast membership + uniqueness | `set` | Average $O(1)$ `in`, auto-dedup. |
| Maintain insertion order + uniqueness | `dict` keys (or `OrderedDict`) | Dict preserves insertion order (Python 3.7+). |
| Allow duplicates | `list` | Keeps all entries; can sort and count. |
| Immutable unique collection | `frozenset` | Hashable, can be dict key/set element. |
| Frequency counting | `collections.Counter` | Stores counts, not just presence. |

## 8. Real-World Applications
- **Graph algorithms**: `visited` sets in DFS/BFS.
- **Access control**: permissions as sets (`{"read", "write"}`) and intersections.
- **Data cleaning**: deduplicating records/IDs quickly.
- **Compilers/linters**: tracking seen symbols/imports.
- **Caching**: maintaining a set of cache keys for fast invalidation checks.

## 9. Practice Questions

### 9.1 Conceptual (5 questions)
1. 🟢 Easy: Why can’t a Python `set` contain a `list` as an element?
2. 🟢 Easy: What is the difference between `{}` and `set()`?
3. 🟡 Medium: When would you choose `frozenset` over `set`?
4. 🟡 Medium: Explain why mutating an object after inserting it into a set can cause bugs.
5. 🔴 Hard: Sets provide average $O(1)$ membership checks. Describe a scenario where performance could degrade and why.

### 9.2 Applied / Scenario-Based (4 questions)
1. 🟢 Easy: You have a list of 10 million user IDs and need to check whether a given ID is present many times. What structure do you use and why?
2. 🟡 Medium: You need to compute “users who purchased in both January and February” and “users who purchased in January but not February.” Which set operations apply?
3. 🟡 Medium: You must deduplicate a list but preserve order. How would you implement it and what is the complexity?
4. 🔴 Hard: Your program uses a set of custom objects. Membership checks are slow and sometimes inconsistent. What methods/properties would you inspect on the class?

### 9.3 Coding / Hands-On (3 tasks)
1. 🟢 Easy: Write a function `unique_count(s: str) -> int` that returns the number of distinct characters in `s`.
2. 🟡 Medium: Given two lists of integers, return elements that appear in both (deduplicated).
3. 🔴 Hard: Implement a function that groups anagrams together using sets/dicts (explain why a set alone is insufficient).

### 9.4 Interview-Style (3 questions)
1. 🟡 Medium: Compare using a `set` vs a `dict` for membership checks. Are they the same internally?
2. 🔴 Hard: How do Python sets handle collisions, and why does that matter for performance?
3. 🟡 Medium: Explain `a < b` for sets in Python. What does it mean?

## 10. Answers & Explanations

<details>
<summary>Answer to 9.1 Q1</summary>

A `list` is **unhashable** because it is mutable; its contents can change, which would change its equality semantics and would make a stable hash impossible. Set elements must be hashable so they can be placed and found reliably in the underlying hash table.

</details>

<details>
<summary>Answer to 9.1 Q2</summary>

`{}` creates an **empty dictionary**. The empty set literal doesn’t exist, so you must use `set()` to create an empty set.

</details>

<details>
<summary>Answer to 9.1 Q3</summary>

Choose `frozenset` when you need an **immutable** set: for example, to use it as a **dictionary key** or as an **element inside another set**, or when you want to guarantee it won’t be modified after creation.

</details>

<details>
<summary>Answer to 9.1 Q4</summary>

Sets rely on an element’s `hash(x)` and equality (`x == y`) to locate it. If you mutate an object in a way that changes its hash or equality behavior after insertion, the set may look in the wrong place during lookup/removal, making the object effectively “disappear” or causing inconsistent results.

</details>

<details>
<summary>Answer to 9.1 Q5</summary>

Performance can degrade if many elements produce **hash collisions**, forcing lots of probing during lookup/insert. In the extreme (pathological hashing or adversarial inputs), operations can approach $O(n)$. Python includes collision-handling strategies and randomized hashing for some types (like `str`) to reduce practical risk, but custom objects with poor `__hash__` can still cause issues.

</details>

<details>
<summary>Answer to 9.2 Q1</summary>

Use a `set` of user IDs. Building it is $O(n)$, and each membership test is average $O(1)$, which is ideal when you will perform many lookups.

</details>

<details>
<summary>Answer to 9.2 Q2</summary>

- “Purchased in both” = intersection: `jan & feb`
- “Purchased in January but not February” = difference: `jan - feb`

These are core set operations implemented efficiently.

</details>

<details>
<summary>Answer to 9.2 Q3</summary>

Use a `seen` set to track membership and a list to preserve order (see section 5.2). Complexity is $O(n)$ time average and $O(n)$ extra space.

</details>

<details>
<summary>Answer to 9.2 Q4</summary>

Inspect the class’s `__hash__` and `__eq__` implementations:
- They must be consistent: if `a == b` then `hash(a) == hash(b)`.
- The attributes used for hashing/equality should be immutable (or not change while the object is in the set).
- Ensure `__hash__` is well-distributed to avoid collisions.

Inconsistencies lead to wrong membership results; poor distribution leads to slow lookups.

</details>

<details>
<summary>Answer to 9.3 Task 1</summary>

```python
def unique_count(s: str) -> int:
    return len(set(s))
```

`set(s)` deduplicates characters; `len(...)` counts distinct characters.

</details>

<details>
<summary>Answer to 9.3 Task 2</summary>

```python
def intersection_dedup(a, b):
    return list(set(a) & set(b))
```

This returns unique common elements. If you need a stable order, sort or preserve order with a `seen` pattern.

</details>

<details>
<summary>Answer to 9.3 Task 3</summary>

A set alone can’t “group” because it only keeps unique elements and discards duplicates; grouping requires mapping a signature to a collection.

```python
from collections import defaultdict

def group_anagrams(words):
    buckets = defaultdict(list)
    for w in words:
        key = tuple(sorted(w))  # hashable signature
        buckets[key].append(w)
    return list(buckets.values())
```

Here we use a dict keyed by a hashable signature; sets can still help (e.g., dedupe words first), but aren’t sufficient for grouping.

</details>

<details>
<summary>Answer to 9.4 Q1</summary>

They are very similar: both are hash-table based and provide average $O(1)$ membership. A `set` stores only keys, while a `dict` stores key→value pairs. If you only need membership/uniqueness, a `set` communicates intent and is slightly more direct.

</details>

<details>
<summary>Answer to 9.4 Q2</summary>

Python sets use open addressing with probing. When two elements map to the same initial slot (collision), the set probes other slots following a probing sequence until it finds either the element or an empty slot. More collisions mean more probes, increasing time per operation; resizing and good hashing keep load factor manageable.

</details>

<details>
<summary>Answer to 9.4 Q3</summary>

For sets, `a < b` means **proper subset**: all elements of `a` are in `b`, and `a` is not equal to `b`. Similarly, `a <= b` means subset (possibly equal).

</details>

## 11. Summary / Cheat Sheet
- `set` = unordered, mutable, unique, hash-based collection.
- Use `set()` for empty set; `{}` is a dict.
- Membership: `x in s` is average $O(1)$.
- Operations: union `|`, intersection `&`, difference `-`, symmetric difference `^`.
- Elements must be **hashable**; don’t mutate hashed identity while inside the set.
- Use `discard` if missing elements are normal.
- Use `frozenset` for immutability / nested sets.

## 12. Further Reading
- Python docs: Built-in Types — `set` and `frozenset` (official documentation)
- Python docs: Data model — `__hash__` and `__eq__`
- CPython implementation notes: `Objects/setobject.c` (source code)
