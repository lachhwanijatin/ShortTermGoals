# C++ Templates & Exception Handling — Interview Cheat Sheet

---

## 1. Function Templates: Overloading vs. Specialization

> **TL;DR:** Explicit function template specialization (`template<>`) is a trap — prefer plain overloading.

Explicit specializations **do not participate in overload resolution**, which leads to surprising behavior.

### Rules of Thumb

| Scenario | Recommendation |
|---|---|
| General case | Use standard function overloading |
| Pointer / reference variants | Write `T*` or `const T&` — this is overloading, not partial specialization |
| Mutable arg | `process(T& ref)` |
| Read-only / r-value arg | `process(const T& ref)` |
| Pointer arg | `process(T* ptr)` |
| Return-type-only deduction | Only valid use case for explicit specialization |

### The L-Value Ambiguity Trap

Do **not** mix `process(T value)` and `process(T& ref)` in the same overload set — passing an l-value causes a compiler ambiguity error.

### The `auto` Return Type Trap

If the primary template uses `auto` for return type deduction (C++14+), specializations **must also use `auto`**:

```cpp
// Primary template
template<typename T1, typename T2>
auto Sum(T1 a, T2 b) { return a + b; }

// FAILS: return type is char*, not auto
// template<> char* Sum<char*, char*>(char* s1, char* s2) { ... }

// WORKS: matches primary template signature exactly
template<> auto Sum<char*, char*>(char* s1, char* s2) { /* returns char* */ }
```

---

## 2. Class Template Specialization

Unlike functions, class templates can be **safely** explicitly or partially specialized.

### Key Concepts

- **Partial Specialization** — customizes behavior for a category of types (e.g., all pointers `T*`) or locks specific parameters (e.g., `<T, int>`).
- **Full Specialization** — customizes behavior for one exact type combination.

### Standard Library Examples

| Example | Verdict | Why |
|---|---|---|
| `std::hash` | ✅ Gold Standard | Primary template is empty; robust full specializations per built-in type |
| `std::vector<bool>` | ⚠️ Historical Flaw | Packs 8 bools into 1 byte; returns a proxy instead of `bool&`, silently breaking generic algorithms |

---

## 3. Type Checking & `std::is_same_v`

> **Under the hood:** Implemented via class template partial specialization — matches `<T, T>` against a fallback `<T, U>`.

### The Strictness Trap

`is_same` is **ruthlessly exact** — `int ≠ const int ≠ int&`.  
Always strip modifiers before comparing deduced template types.

### Stripping Type Modifiers

| Standard | Tool | Strips |
|---|---|---|
| C++20 | `std::remove_cvref_t<T>` | `const`, `volatile`, references |
| C++14 | `std::decay_t<T>` | Same, **plus** decays arrays → pointers, functions → function pointers |

```cpp
template <typename T>
void process(T&& arg) {
    using CleanType = std::remove_cvref_t<T>; // C++20
    if constexpr (std::is_same_v<CleanType, int>) {
        // Safe comparison — modifiers stripped
    }
}
```

---

## 4. `typename` vs `class`

Interchangeable in `template <typename T>` / `template <class T>` declarations, but have **strict rules elsewhere**.

| Keyword | When Mandatory |
|---|---|
| `typename` | Resolving **dependent names** (nested type definitions inside a template type) |
| `class` | Structurally defining a class/struct; template template parameters (pre-C++17) |

> The compiler assumes `T::something` is a static value/variable by default. Use `typename` to explicitly declare it is a nested type.

```cpp
template <typename Container>
void iterate(Container& c) {
    typename Container::value_type init{}; // 'typename' strictly required
}
```

---

## 5. Exception Handling

### Core Rules

**Rule 1 — Throw by value, catch by `const` reference.**

```cpp
catch (const std::exception& e) { ... }
```

Catching by value causes **object slicing** (derived exception data is lost) and unnecessary copying.

**Rule 2 — Never throw primitives.** Use `<stdexcept>` types (`std::invalid_argument`, `std::runtime_error`, etc.) — they guarantee `.what()`.

### Custom Exceptions

```cpp
class DatabaseException : public std::exception {
    std::string msg;
public:
    DatabaseException(const std::string& m) : msg(m) {}
    const char* what() const noexcept override { return msg.c_str(); }
    //                        ^^^^^^^^ never let error retrieval itself throw
};
```

---

### Under the Hood: Memory & Stack Unwinding

| Concept | Detail |
|---|---|
| **When is memory allocated?** | Exactly when `throw` executes — not before; if the line is skipped, zero memory is used |
| **Where is the exception object stored?** | A special compiler-managed region — **not** the stack, **not** the heap — so it survives stack destruction |
| **Stack Unwinding** | On `throw`, normal execution halts; runtime walks backward up the call stack, calling destructors for all local objects (RAII) |

---

### Multiple Simultaneous Exceptions

| Scenario | Outcome |
|---|---|
| **Multithreading** | ✅ Safe — Thread A and B can throw/catch independently; exception memory is per-thread |
| **Exception Nesting** | ✅ Safe — Bundle a low-level error inside a high-level error via `std::throw_with_nested`; extract with `std::rethrow_if_nested` |
| **Throwing inside a destructor during unwinding** | ☠️ Fatal — If Exception B is thrown while unwinding for Exception A, C++ cannot resolve priority; `std::terminate()` is called immediately |

> **Golden Rule: Never throw from a destructor.**

---

## 6. The C++ Runtime (RTE)

Unlike Java (JVM) or Python, C++ has **no separate background runtime process**. The runtime is compiled, optimized boilerplate (`libstdc++`, `libc++`) **embedded directly into your binary**.

### What the Runtime Handles

| Responsibility | Description |
|---|---|
| **Setup / Teardown** | Runs initialization before `main()`, cleans up globals and flushes streams after `main()` exits |
| **Exception Plumbing** | Provides hidden functions (e.g., `__cxa_allocate_exception`) for allocating exception memory and performing CPU register tracking for stack unwinding |
| **Memory Management** | Translates `new` / `delete` into OS-level allocation requests |
| **RTTI** | Manages hidden type metadata enabling `dynamic_cast` to resolve types at runtime |
