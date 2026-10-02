# Day 08 - Python Quick Q&A

Rapid-fire short questions with short answers — good for a quick refresher pass, not deep scenarios.

---

**Q1. `is` vs `==`?**

`==` compares values for equality; `is` compares object identity (same memory reference). Never use `is` to compare numbers/strings for value equality.

**Q2. Why does `a = [1, 2]; b = [1, 2]; print(a is b)` print `False` but `a == b` prints `True`?**

They're two distinct list objects with equal contents — `==` checks value equality, `is` checks whether they're literally the same object.

**Q3. Shallow copy vs deep copy?**

`copy.copy()` duplicates the outer container but nested objects are still shared references; `copy.deepcopy()` recursively duplicates everything, so nested mutations don't affect the original.

**Q4. What's wrong with `new_list = old_list` when you want an independent copy?**

It doesn't copy anything — both names point to the same list object, so mutating one mutates the other. Use `old_list.copy()` or `old_list[:]`.

**Q5. Classic closure bug: what does this print?**

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])
```
`[2, 2, 2]` — the lambda captures the variable `i` by reference, not its value at creation time, and `i` is `2` by the time the lambdas run. Fix with a default argument: `lambda i=i: i`.

**Q6. `*args` vs `**kwargs`?**

`*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict.

**Q7. What does unpacking `*` do in `first, *rest = [1, 2, 3, 4]`?**

`first = 1` and `rest = [2, 3, 4]` — it captures the remaining elements into a list.

**Q8. List comprehension vs generator expression — key difference?**

A list comprehension (`[x for x in ...]`) builds the entire list in memory immediately; a generator expression (`(x for x in ...)`) yields items lazily one at a time, using constant memory.

**Q9. When would a generator be the wrong choice?**

When you need to iterate the data multiple times or need random access/indexing — a generator is exhausted after one pass and can't be re-read or sliced.

**Q10. What does the `with` statement guarantee that manual `open()`/`close()` doesn't?**

The file (or resource) is closed automatically via `__exit__`, even if an exception is raised inside the block — manual close can be skipped if an error happens first.

**Q11. `try/except/else/finally` — when does each run?**

`try` runs first; `except` runs only if an exception matched occurs; `else` runs only if no exception occurred; `finally` always runs regardless of exceptions, typically for cleanup.

**Q12. What's the problem with a bare `except:`?**

It catches everything, including `KeyboardInterrupt` and `SystemExit`, silently hiding bugs and making debugging harder — catch specific exception types instead.

**Q13. What does `raise SomeError("msg") from original_exc` do?**

Explicitly chains exceptions, preserving the original traceback/cause in `__cause__` so debugging shows both the new error and what originally triggered it.

**Q14. `@staticmethod` vs `@classmethod`?**

`@staticmethod` takes no implicit first argument and behaves like a plain function namespaced in the class; `@classmethod` takes `cls` as the first argument and can access/modify class-level state or act as an alternate constructor.

**Q15. What is the GIL and what does it actually block?**

The Global Interpreter Lock ensures only one thread executes Python bytecode at a time, so CPU-bound multithreading doesn't speed up pure-Python code; I/O-bound work still benefits from threads since the GIL is released during I/O waits.

**Q16. When would you use `multiprocessing` instead of `threading`?**

For CPU-bound work (heavy computation) where you need true parallelism across cores, bypassing the GIL — each process has its own interpreter and memory space.

**Q17. What does `dict.get(key, default)` avoid compared to `dict[key]`?**

A `KeyError` if the key is missing — `.get()` returns the default (or `None`) instead of raising.

**Q18. What does `dict.setdefault(key, default)` do that `.get()` doesn't?**

If the key is missing, it both inserts the default into the dict and returns it; `.get()` only returns a value without modifying the dict.

**Q19. Why is a `set` faster than a `list` for membership checks (`in`)?**

A `set` (like a `dict`) uses hashing for average O(1) lookups; a `list` requires a linear O(n) scan.

**Q20. Why can't you put a `list` inside a `set` or use it as a `dict` key?**

Lists are mutable and unhashable; sets/dict keys require hashable (effectively immutable) elements — use a `tuple` instead.

**Q21. What's the output and why?**

```python
print(0.1 + 0.2 == 0.3)
```
`False` — binary floating-point can't represent 0.1/0.2/0.3 exactly, so tiny rounding errors make the sum slightly off; compare with `math.isclose()` or use `Decimal` for exact money math.

**Q22. Why is repeated string concatenation in a loop (`s += x`) inefficient, and what's the fix?**

Strings are immutable, so each `+=` creates a brand-new string and copies everything so far (O(n²) overall); build a list and `"".join(list)` once instead.

**Q23. What does `enumerate(items, start=1)` do?**

Yields `(index, item)` pairs starting the index counter at `1` instead of the default `0`.

**Q24. `sorted(items, key=...)` vs `items.sort(key=...)`?**

`sorted()` returns a new list and leaves the original unchanged (works on any iterable); `.sort()` mutates the list in place and returns `None`.

**Q25. What's the risk of using a mutable default argument like `def f(items=[])`?**

The default list is created once at function definition time and shared/mutated across all calls that don't pass their own argument — leads to unexpected accumulation. Use `None` and create the list inside the function instead.

**Q26. What does a decorator actually do under the hood?**

`@my_decorator\ndef f(): ...` is sugar for `f = my_decorator(f)` — it wraps the original function and returns a replacement, commonly used for logging, timing, caching, or access control.

**Q27. What does `functools.lru_cache` do and when is it unsafe to use?**

Caches a function's return value by its arguments to avoid recomputation; unsafe for functions with side effects, unhashable arguments, or when inputs/outputs can change over time (e.g., wrapping a function that reads live data).

**Q28. What's the MRO (Method Resolution Order) and why does it matter in multiple inheritance?**

The order Python searches base classes for an attribute/method (C3 linearization); it determines which parent's method actually runs when multiple parents define the same name — check it with `ClassName.__mro__`.

**Q29. `__init__` vs `__new__`?**

`__new__` actually creates and returns the new instance (rarely overridden, used for immutable types or singletons); `__init__` initializes an already-created instance and returns `None`.

**Q30. What's the walrus operator (`:=`) useful for?**

Assigning a value as part of an expression, avoiding a redundant call — e.g., `while (chunk := file.read(1024)):` reads and checks in one line.

**Q31. Why might `json.dumps()` fail on a dict containing a `datetime` object?**

`datetime` isn't natively JSON-serializable; you need a custom `default=` serializer function or convert to an ISO string (`.isoformat()`) first.

**Q32. Naive vs timezone-aware `datetime` objects — why does mixing them raise an error?**

A naive `datetime` has no timezone info and an aware one does; Python refuses to compare/subtract them directly because the offset needed to make them comparable is unknown — localize or convert both to the same awareness first.

**Q33. What does `assert` do in production code, and why is it risky for validation?**

`assert` raises `AssertionError` if the condition is false, but assertions are stripped out entirely when Python runs with the `-O` (optimize) flag — never use `assert` for real input validation or security checks.

**Q34. What's the difference between a `tuple` and a `list` besides mutability?**

Tuples are hashable (if contents are hashable) and slightly more memory/CPU efficient for fixed collections, so they can be used as dict keys or set members; lists can't.

**Q35. What does a `dataclass` give you over a plain class?**

Auto-generated `__init__`, `__repr__`, and `__eq__` based on declared fields, reducing boilerplate for simple data-holding classes.
