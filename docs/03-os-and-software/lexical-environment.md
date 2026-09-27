# How the Lexical Environment Works (Programming)

> **Level:** Advanced · **Related:** [Code Compilation](code-compilation.md) · [Software/Apps](software-and-apps.md)

## 1. Definitions

- **Scope**: the region of source code where a name (variable, function) is visible.
- **Lexical (static) scoping**: a name's meaning is determined by **where it is written in the source**, not by who calls the code at run time. JavaScript, Python, C++, Rust, Java — nearly all modern languages — are lexically scoped.
- **Dynamic scoping** (the alternative): a name resolves by walking the **call stack** at run time (old Lisps, Bash variables, Emacs Lisp by default).
- **Lexical Environment** (ECMAScript spec term): the run-time data structure that implements lexical scoping. It consists of:
  1. an **Environment Record** — a mapping of identifiers to values/bindings in this scope, and
  2. an **[[OuterEnv]]** reference — a pointer to the lexically enclosing environment (null at global).

Looking up a name = check the current record → follow `outer` → … → global → `ReferenceError`. This linked list is the **scope chain**.

```
        Global Environment  { counter: <fn>, console: ..., ... }  outer: null
               ▲
        makeCounter() Env   { count: 0 }                            outer: Global
               ▲
        increment() Env     { }                                     outer: makeCounter Env
```

## 2. Lexical vs dynamic, side by side

```js
const x = "global";
function show() { console.log(x); }       // x resolved where show is DEFINED
function run() { const x = "local"; show(); }
run();   // JavaScript (lexical): "global".  A dynamically scoped language would print "local".
```

Bash is dynamically scoped for `local` variables:

```bash
x=global
show() { echo "$x"; }
run() { local x=local; show; }
run   # prints "local"
```

## 3. JavaScript in depth (ECMAScript model)

### 3.1 Kinds of Environment Records

| Record | Created for | Holds |
|---|---|---|
| **Declarative** | Blocks `{}`, functions, `catch`, modules | `let`, `const`, `class`, function params, inner function declarations |
| **Function** (declarative subtype) | Each function call | params, `arguments`, `this` binding, `super`/`new.target` |
| **Object** | `with` statements, global object part | Bindings are properties of an object |
| **Global** | Once per realm | Object record (globalThis: `var`, function decls) + declarative record (top-level `let/const/class`) |
| **Module** | Each ES module | Top-level bindings + immutable live **import bindings** |

### 3.2 Execution contexts
Each running function has an **execution context** with two environment pointers:
- **VariableEnvironment** — where `var` and function declarations live (function-scoped).
- **LexicalEnvironment** — current block's environment (changes as you enter `{}` blocks with `let/const`).

### 3.3 Creation phase: hoisting and the Temporal Dead Zone

Before a scope's code runs, the engine **instantiates all its bindings**:
- `var` → created and initialized to `undefined`.
- Function declarations → created and initialized to the function object.
- `let`/`const`/`class` → created but **uninitialized**: accessing them before the declaration line throws — the **Temporal Dead Zone (TDZ)**.

```js
console.log(a);     // undefined          (var hoisted + initialized)
console.log(hello()); // "hi"             (function declaration fully hoisted)
console.log(b);     // ReferenceError: Cannot access 'b' before initialization (TDZ)
var a = 1;
let b = 2;
function hello() { return "hi"; }
```

### 3.4 Closures: functions capture environments, not values

When a function object is created, it stores its defining environment in an internal slot **`[[Environment]]`**. When called, a new environment is created whose `outer` is that saved one. As long as the function is reachable, so is that environment (it can't be garbage-collected).

```js
function makeCounter() {
  let count = 0;                         // lives in makeCounter's environment
  return {
    inc: () => ++count,                  // both closures share the SAME environment
    get: () => count,
  };
}
const c1 = makeCounter(), c2 = makeCounter();
c1.inc(); c1.inc(); c2.inc();
console.log(c1.get(), c2.get());          // 2 1  — each call made a fresh environment
```

### 3.5 The classic loop bug — and why `let` fixes it

```js
var fns = [];
for (var i = 0; i < 3; i++) fns.push(() => i);
console.log(fns.map(f => f()));   // [3, 3, 3] — one function-scoped `i` shared by all closures

const fns2 = [];
for (let j = 0; j < 3; j++) fns2.push(() => j);
console.log(fns2.map(f => f()));  // [0, 1, 2] — spec creates a NEW environment per iteration
                                  // (CreatePerIterationEnvironment copies j each loop)
```

### 3.6 `this` is NOT lexical (except in arrow functions)

`this` is bound per call (how the function is invoked), stored in the Function Environment Record. **Arrow functions** have no own `this` — they resolve it lexically through the scope chain like any variable.

```js
const obj = {
  name: "obj",
  regular() { return [1].map(function () { return this?.name; }); }, // [undefined] (strict)
  arrow()   { return [1].map(() => this.name); },                    // ["obj"] — lexical this
};
```

### 3.7 Engine reality (V8)
The spec model is conceptual. V8's parser performs **scope analysis** at compile time:
- Variables **not captured** by any closure live in registers/stack slots — no heap allocation.
- Captured variables are placed in a heap-allocated **Context** object; closures point to it.
- `eval` and `with` defeat this analysis (the engine must keep everything in a dynamic lookup), which is one reason they're slow and discouraged.
- Memory leak pattern: a long-lived closure keeps a big object alive because it shares a Context with another closure that referenced it.

Inspect in Chrome DevTools: set a breakpoint → **Scope** panel shows `Local`, `Block`, `Closure (makeCounter)`, `Script`, `Global`.

## 4. Python: LEGB and the compiler's role

Python resolves names by **LEGB**: **L**ocal → **E**nclosing function scopes → **G**lobal (module) → **B**uiltins.

Crucially, the **compiler** decides at compile time which scope each name belongs to: any assignment inside a function makes the name **local to the whole function** (unless declared `global`/`nonlocal`).

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        print(x)            # 'enclosing' — resolved via closure cell
    inner()
outer()

def broken():
    print(x)                # UnboundLocalError! assignment below makes x local
    x = 1                   # (Python's analogue of the TDZ)

def counter():
    n = 0
    def inc():
        nonlocal n          # rebind the enclosing variable, not create a local
        n += 1
        return n
    return inc
c = counter(); c(); print(c())          # 2
```

Under the hood (CPython):

```python
import dis
def make():
    n = 0
    def inc():
        nonlocal n; n += 1; return n
    return inc
inc = make()
print(inc.__code__.co_freevars)        # ('n',)  — free variables captured
print(inc.__closure__[0].cell_contents)  # 0     — the shared "cell" object
dis.dis(inc)                            # LOAD_DEREF / STORE_DEREF on the cell
```

- Local variables → `LOAD_FAST` (array slot in the frame, very fast).
- Captured variables → **cell objects** (`LOAD_DEREF`), shared between the outer frame and closures.
- Globals → dict lookup (`LOAD_GLOBAL`, specialized/cached in 3.11+).
- Blocks (`if`, `for`) do **not** create scopes in Python; comprehensions and functions do (and since 3.12, comprehensions are inlined but still isolate their loop variable).

Late-binding closure gotcha (same as JS `var`):

```python
fns = [lambda: i for i in range(3)]
print([f() for f in fns])               # [2, 2, 2] — all share the comprehension's i cell
fns = [lambda i=i: i for i in range(3)] # default arg captures current value
print([f() for f in fns])               # [0, 1, 2]
```

## 5. C++: lexical scope at compile time, explicit captures

C++ has block scope, namespace scope, class scope, and function-parameter scope — all resolved **entirely at compile time**; no runtime environment objects exist. Local variables live on the stack and die at scope end.

Lambdas make capture **explicit**, and the programmer chooses **by value** or **by reference**:

```cpp
#include <functional>
#include <iostream>
#include <memory>

std::function<int()> make_counter() {
    int count = 0;
    // return [&count] { return ++count; };  // BUG: dangling reference — count dies on return
    return [count]() mutable { return ++count; };   // copy lives inside the closure object
}

std::function<int()> make_shared_counter() {
    auto count = std::make_shared<int>(0);          // heap state shared by copies
    return [count] { return ++*count; };
}

int main() {
    auto c = make_counter();
    c(); std::cout << c() << "\n";                  // 2
    int x = 10;
    auto byVal = [x] { return x; };
    auto byRef = [&x] { return x; };
    x = 20;
    std::cout << byVal() << " " << byRef() << "\n"; // 10 20
}
```

A lambda compiles into an anonymous **class** ("closure type") whose data members are the captures and whose `operator()` is the body — the compile-time equivalent of a JS function + `[[Environment]]`.

Name lookup rules worth knowing: **shadowing** (inner declaration hides outer), **unqualified lookup** walks enclosing scopes outward, **ADL** (argument-dependent lookup) also searches namespaces of argument types, and templates have **two-phase lookup**.

## 6. Comparison

| Aspect | JavaScript | Python | C++ |
|---|---|---|---|
| Block scope | `let`/`const`/`class` yes; `var` no | No (function/module/class/comprehension only) | Yes |
| Hoisting | `var` → undefined; functions fully; `let/const` TDZ | Names are local for the whole function (UnboundLocalError) | No — must declare before use |
| Closures capture | Environment (by reference) | Cells (by reference) | Explicit: by value or by reference |
| Rebind outer var | Just assign | `nonlocal` / `global` | Capture by reference |
| Lifetime of captured vars | GC keeps env alive | Cells refcounted/GC'd | Programmer's responsibility |
| Resolution time | Mostly compile-time analysis; spec'd as runtime chain | Compile-time scope classification, runtime lookup | Compile time |

## 7. Why it matters in practice

1. **Encapsulation / private state** — the module pattern, factory functions, React hooks (`useState` values are captured per render — the "stale closure" problem in `useEffect` dependencies).
2. **Callbacks & async** — closures carry context into event handlers and promises.
3. **Memory** — closures can unintentionally retain large objects.
4. **Performance** — engines optimize well-behaved static scopes; `eval`/`with` break optimizations.
5. **Security** — scope isolation underpins sandboxing patterns (ES modules, `Object.freeze`, SES/Compartments).

## Further reading
- ECMAScript® Language Specification, §9.1 *Environment Records*, §9.4 *Execution Contexts*
- Kyle Simpson, *You Don't Know JS Yet: Scope & Closures*
- Python Language Reference, §4.2 *Naming and binding*; PEP 227 (nested scopes), PEP 3104 (`nonlocal`)
- cppreference: *Lambda expressions*, *Scope*, *Unqualified name lookup*
