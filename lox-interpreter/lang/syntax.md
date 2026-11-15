# Chapter 3 — The Lox Language

> Study notes for Chapter 3 of _Crafting Interpreters_: a tour of Lox, the language the rest of the book implements.

## Table of Contents

0. [The Big Picture](#0-the-big-picture)
1. [Hello, Lox: Syntax and Design](#1-hello-lox-syntax-and-design)
2. [High-Level Language, Dynamic Typing, and Memory Management](#2-high-level-language-dynamic-typing-and-memory-management)
3. [The Four Built-in Data Types](#3-the-four-built-in-data-types)
4. [Expressions: Things That Produce Values](#4-expressions-things-that-produce-values)
5. [Statements: Things That Do Something](#5-statements-things-that-do-something)
6. [Variables and State](#6-variables-and-state)
7. [Control Flow](#7-control-flow)
8. [Functions](#8-functions)
9. [Closures](#9-closures)
10. [Classes and Objects](#10-classes-and-objects)
11. [Inheritance](#11-inheritance)
12. [Primitives Are Not Full Objects](#12-primitives-are-not-full-objects)
13. [The Standard Library](#13-the-standard-library)
14. [Design Note: Expressions vs. Statements](#14-design-note-expressions-vs-statements)
15. [Implementation Consequences to Remember](#15-implementation-consequences-to-remember)
16. [Common Misunderstandings and Traps](#16-common-misunderstandings-and-traps)
17. [Practical Examples: Predict the Output](#17-practical-examples-predict-the-output)
18. [Chapter Challenges](#18-chapter-challenges)
19. [How to Study This Chapter](#19-how-to-study-this-chapter)
20. [Connections to Rust and C Systems Programming](#20-connections-to-rust-and-c-systems-programming)
21. [Self-Test](#21-self-test)
22. [Final Mental Model](#22-final-mental-model)

---

## 0. The Big Picture

Chapter 3 is a tour of **Lox**, the small programming language that the rest of _Crafting Interpreters_ implements.

It is not yet a chapter about writing the scanner, parser, interpreter, compiler, virtual machine, or garbage collector. Instead, it describes what the finished language should look like from a programmer's point of view.

Think of it as a **language requirements document**:

- What values can a Lox program contain?
- What operations can it perform?
- What does an expression mean?
- What is a statement?
- How do variables and scopes work?
- How does control flow work?
- How are functions represented and called?
- What does it mean for a function to be a closure?
- How do classes, objects, methods, and inheritance work?
- What is built into the language?

**Why this matters:** an interpreter is not just code that reads text and executes it. It must implement _precise rules_. If the language says only `false` and `nil` are falsey, your interpreter must not accidentally use the host language's truthiness rules. If functions are closures, a function may need to keep variables alive after the function that created them has returned.

The book builds two implementations of the same language:

| Implementation | Strategy                 | Host language |
| -------------- | ------------------------ | ------------- |
| **jlox**       | Tree-walk interpreter    | Java          |
| **clox**       | Bytecode virtual machine | C             |

The strategies differ, but observable behavior should be identical. This is a useful separation:

> **Semantics** describe _what_ programs mean. **Implementation** describes _how_ the runtime makes it happen.

### Lox at a glance

- High-level scripting language
- C-family syntax: braces, semicolons, familiar operators
- Dynamically typed
- Automatically managed memory
- Lexical (block) scope
- First-class functions and closures
- Classes, instances, methods, and single inheritance
- Small standard library

It resembles JavaScript in surface syntax, borrows scoping ideas from Scheme, and its C implementation draws on Lua's compact runtime design.

---

## 1. Hello, Lox: Syntax and Design

A first Lox program:

```lox
// Your first Lox program!
print "Hello, world!";
```

`//` begins a line comment; the semicolon terminates the statement.

> [!IMPORTANT]
> `print` is a **language statement**, not a function call.
>
> ```lox
> print "Hello";    // ✅ the Lox form
> print("Hello");   // ❌ not the Lox form
> ```

**Why C-like syntax?** Familiar syntax reduces how much the reader has to learn, and the book's implementations use Java and C, which share that syntax. Lox deliberately avoids C's complicated type-declaration syntax because it has no static type system.

### Syntax vs. semantics

| Term          | Meaning                                      |
| ------------- | -------------------------------------------- |
| **Syntax**    | The shape/grammar of valid source code       |
| **Semantics** | What valid source code _means_ when executed |

`a + b` is syntax. Whether it adds two numbers or joins two strings is semantics.

> [!WARNING]
> Familiar syntax does not imply familiar semantics. Never assume Lox behaves like JavaScript or C just because it looks like them.

---

## 2. High-Level Language, Dynamic Typing, and Memory Management

### 2.1 Dynamic typing

A variable is not permanently bound to a type. It can hold values of different types at different moments:

```lox
var value = 42;
print value;      // 42

value = "hello";
print value;      // hello

value = true;
print value;      // true
```

The _value_ has a type; the variable name is simply a place through which the current value is accessed.

Type errors are therefore discovered at runtime:

```lox
print 10 / "apple";   // runtime error: operands must be numbers
```

|          | Static typing                                                               | Dynamic typing                                            |
| -------- | --------------------------------------------------------------------------- | --------------------------------------------------------- |
| **When** | Before execution                                                            | During execution                                          |
| **How**  | A type checker rejects invalid operations using the language's static rules | The runtime inspects actual values when an operation runs |

> [!NOTE]
> Dynamic typing does **not** mean "there are no types." Lox has types; checking them is a runtime responsibility.

**Why dynamic typing for Lox?** It keeps the language and implementation smaller. A static type system would add substantial machinery and distract from the book's goal: implementing interpreters and VMs.

**Implementation connection.** A runtime value must carry enough information to identify its type. Conceptually:

```text
Number(42)   String("hello")   Boolean(true)   Nil
```

In Rust, an educational representation might be:

```rust
enum Value {
    Number(f64),
    Bool(bool),
    String(String),
    Nil,
}
```

This is an illustration, not the book's Java/C representation. In C, a similar design uses a **tagged union**: a tag says which variant is active, and a union stores the payload.

> **Invariant:** never interpret a string payload as a number just because both occupy machine memory.

### 2.2 Automatic memory management

The programmer does not manually free strings, functions, classes, instances, or other objects. Two broad approaches:

| Approach               | How it works                                                                                | Weakness                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Reference counting** | Each object tracks how many references point to it; reclaimed when the count hits zero      | Cycles: if A → B and B → A, counts never reach zero even when unreachable |
| **Tracing GC**         | Start from known live references (_roots_), follow references, reclaim anything unreachable | More complex to implement                                                 |

Tracing collectors reclaim unreachable cycles because they reason about _reachability from roots_, not local counts. The book eventually implements a tracing GC in clox — language features like closures, strings, classes, and instances create runtime objects whose lifetimes must be managed correctly.

**Key terms**

- **Allocation** — obtaining memory for an object.
- **Reachability** — whether an object can be reached by following references from roots.
- **Garbage** — allocated memory that is no longer reachable and can be reclaimed.
- **Root** — a reference the runtime knows is live (a global, an active stack value, etc.).

> [!WARNING]
> Automatic memory management ≠ infinite memory. A GC'd program can still allocate excessively or retain objects longer than necessary.

---

## 3. The Four Built-in Data Types

Lox keeps its value universe deliberately small.

### 3.1 Boolean

Exactly two literals: `true` and `false`.

```lox
print true;
print false;
print !true;   // false
```

Unlike languages that use integers as truth values, Lox has a dedicated Boolean type.

### 3.2 Number

One numeric type: **double-precision floating point**.

```lox
1234
12.34
0
-42
```

There is no separate integer type. Consequences:

- Integers and decimals share one type.
- Floating-point precision rules apply.
- Some integers cannot be represented exactly as a double.
- Output formatting requires care — internally `2.0`, but it should print as `2`. Numeric storage and user-facing stringification are separate concerns.

> [!NOTE]
> A double is not arbitrary-precision decimal. Some decimal fractions cannot be represented exactly in binary — a property of the representation, not of Lox.

### 3.3 String

Strings are enclosed in double quotes:

```lox
"hello"
""
"123"
"A string with spaces"
```

The empty string is valid. `"123"` is not the number `123`:

```lox
print 123 == "123";   // false
```

Strings look simple but raise real implementation questions:

- What encoding is used?
- Is a "character" a byte, a Unicode code point, or a grapheme cluster?
- How is length measured?
- How are strings allocated, stored, compared, and reclaimed?

For example, `é` can be encoded several ways depending on normalization, and a UTF-8 string's byte length is not always its number of user-perceived characters. You don't need to solve Unicode in Chapter 3 — just recognize that a string is a **runtime data structure**, not merely "characters between quotes."

### 3.4 Nil

Lox spells its no-value value `nil`. Other languages use `null` or `None`; don't assume identical semantics.

```lox
var result;
print result;   // nil
```

A variable declared without an initializer defaults to `nil`. Lox's `nil` is a _Lox value_ — not a C null pointer or Java null reference, though the concepts are related.

### Summary

| Type    | Example values  |
| ------- | --------------- |
| Boolean | `true`, `false` |
| Number  | `1234`, `12.34` |
| String  | `"hello"`, `""` |
| Nil     | `nil`           |

---

## 4. Expressions: Things That Produce Values

An **expression** is code that evaluates to a value:

```lox
1 + 2
"hello" + " world"
10 > 5
!false
(min + max) / 2
```

Expressions nest; the inner ones are **subexpressions**. The values an operator works on are its **operands**.

### 4.1 Arithmetic operators

| Operator | Meaning          |
| -------- | ---------------- |
| `a + b`  | addition         |
| `a - b`  | subtraction      |
| `a * b`  | multiplication   |
| `a / b`  | division         |
| `-a`     | numeric negation |

```lox
print 2 + 3;      // 5
print 8 - 3;      // 5
print 4 * 5;      // 20
print 20 / 4;     // 5
print -10;        // -10
print -(2 + 3);   // -5
```

Binary operators have two operands:

```text
2   +   3
^       ^
|       |
left    right operand
```

The minus sign appears in two positions:

- `8 - 3` — binary / **infix** subtraction
- `-3` — unary / **prefix** negation

_Infix_: between operands. _Prefix_: before the operand. _Postfix_: after the operand.

### 4.2 The special `+` operator

Most arithmetic operators accept only numbers. `+` has two legal uses:

| Left          | Right  | Result            |
| ------------- | ------ | ----------------- |
| Number        | Number | numeric addition  |
| String        | String | concatenation     |
| anything else |        | **runtime error** |

```lox
print 10 + 20;              // 30
print "Rust" + " language"; // Rust language
print "age:" + 21;          // ❌ runtime error
```

> [!IMPORTANT]
> Lox does **not** implicitly convert `21` to `"21"`. The `+` evaluator must inspect both operand types and pick one of the two legal operations, or report an error.

### 4.3 Comparison operators

`<` `<=` `>` `>=` — numbers only; they produce Booleans.

```lox
print 2 < 3;    // true
print 3 <= 3;   // true
print 9 > 10;   // false
print 9 >= 9;   // true
```

Comparing a number with a string is a runtime type error.

### 4.4 Equality operators

`==` and `!=` accept values of any type, but **values of different types are never equal**.

```lox
print 1 == 1;         // true
print 1 == 2;         // false
print "cat" == "cat"; // true
print "cat" != "dog"; // true
print 123 == "123";   // false
print true == 1;      // false
```

| Language           | `1 == "1"`    |
| ------------------ | ------------- |
| JavaScript (loose) | can be `true` |
| Lox                | `false`       |

> [!WARNING]
> Don't inherit the host language's equality. In Python, `True == 1` is `True` — that must not leak into a Lox interpreter. Compare by Lox type tags and Lox rules.

### 4.5 Logical NOT

```lox
print !true;    // false
print !false;   // true
```

`!` negates _truthiness_ (see §4.7), which is not the same as checking for a literal Boolean.

### 4.6 Logical AND and OR

Spelled as words: `and`, `or`.

```lox
print true and true;    // true
print true and false;   // false
print false or true;    // true
print false or false;   // false
```

The surprise: **they return an operand, not necessarily a Boolean.**

- **`and`** — if the left operand is falsey, return it; otherwise evaluate and return the right.
- **`or`** — if the left operand is truthy, return it; otherwise evaluate and return the right.

### 4.7 Truthiness

> **`false` and `nil` are falsey. Everything else is truthy.**

That includes `0`, `0.0`, `""`, `"hello"`, and `true`.

```lox
print !nil;       // true
print !false;     // true
print !0;         // false
print !"";        // false
print !"hello";   // false
```

This surprises Python programmers, where `0` and `""` are falsey. Never delegate Lox truthiness to the host language — implement a dedicated function:

```text
isTruthy(value):
    if value is nil:            return false
    if value is Boolean(false): return false
    otherwise:                  return true
```

This tiny rule governs `if`, loops, `!`, `and`, and `or`.

### 4.8 Short-circuit evaluation

The right side is **not evaluated** when it cannot affect the result.

- **AND:** left is falsey → return it, skip the right.
- **OR:** left is truthy → return it, skip the right.

```lox
fun dangerous() {
  print "This should not execute";
  return true;
}

print false and dangerous();
print true or dangerous();
```

Neither call runs, so the message is never printed.

**Why this is more than an optimization:** the right side may have side effects, raise a runtime error, or call a function. Skipping it changes observable behavior.

Operand-returning examples:

```lox
print "hello" or "fallback";  // hello
print nil or "fallback";      // fallback
print 0 and "next";           // next  (0 is truthy)
print false and "next";       // false
```

> [!IMPORTANT]
> Do not implement `and`/`or` as eager binary operators that always evaluate both sides. Evaluation order is part of the semantics.

### 4.9 Precedence, associativity, and grouping

- **Precedence** decides which operator groups first.
- **Associativity** decides how operators of equal precedence group.

```lox
print 2 + 3 * 4;     // 2 + (3 * 4) = 14
print (2 + 3) * 4;   // 20
```

Subtraction is left-associative:

```text
10 - 3 - 2   means   (10 - 3) - 2 = 5
             not     10 - (3 - 2) = 9
```

Lox follows familiar C-family precedence; later parsing chapters formalize it. The parser must encode these rules — token order alone does not determine evaluation.

**Deliberately omitted** from base Lox: bitwise operators, shifts, modulo (`%`), and the ternary `?:`. These are possible extensions, not part of the language.

---

## 5. Statements: Things That Do Something

|                | Primarily…           |
| -------------- | -------------------- |
| **Expression** | produces a _value_   |
| **Statement**  | produces an _effect_ |

Effects include printing output, changing a variable, executing a block, choosing a branch, repeating code, and returning from a function.

### 5.1 Print statement

```lox
print "Hello, world!";
```

Evaluates the expression and displays the result.

**Why is `print` built into the grammar?** Pragmatism: the early interpreter can show output before the language has functions, name lookup, or a standard library. It's a teaching convenience, not a recommendation for real languages.

### 5.2 Expression statement

An expression followed by a semicolon is a statement. Its value is discarded:

```lox
1 + 2;
```

Useful when the expression has a side effect:

```lox
breakfast = "beignets";
```

### 5.3 Blocks

A block groups statements and **introduces a scope**:

```lox
{
  print "First";
  print "Second";
}
```

```lox
{
  var secret = "inside";
  print secret;
}
// secret is not available here.
```

Braces carry semantic meaning — they are not just visual indentation.

---

## 6. Variables and State

Declare with `var`:

```lox
var username = "subh";
var attempts = 3;
var enabled = true;
```

No initializer means `nil`:

```lox
var result;
print result;   // nil
```

Read by name; assign with `=`:

```lox
var language = "Lox";
print language;

language = "Rust";
print language;   // Rust
```

Declaration and assignment are distinct:

```lox
var count = 10;   // introduces a variable
count = 11;       // changes an existing variable
```

Dynamic typing lets a variable change type:

```lox
var item = 10;
item = "ten";
```

### Lexical scope and shadowing

Lox uses **lexical scope**: the position of declarations in source code determines which variable a name refers to.

```lox
var name = "outer";

{
  var name = "inner";
  print name;   // inner
}

print name;     // outer
```

The inner declaration _shadows_ the outer one within the block. Exact resolution rules come later (statements/state and resolving chapters).

**Implementation connection:** environments are commonly mappings from names to values, each linked to an enclosing environment. A block creates a new environment whose parent is the previous one.

> [!WARNING]
> Don't implement scope as one global dictionary if you want correct block-local variables and closures.

---

## 7. Control Flow

Lox includes `if`/`else`, `while`, and C-style `for`.

### 7.1 `if` / `else`

```lox
if (condition) {
  print "yes";
} else {
  print "no";
}
```

If the condition is truthy, run the first branch; otherwise the `else` branch (which is optional). Remember Lox truthiness:

```lox
if (0) {
  print "zero is truthy in Lox";   // this runs
}
```

### 7.2 `while`

```lox
var count = 1;

while (count < 4) {
  print count;
  count = count + 1;
}
// Output: 1 2 3
```

Execution:

1. Evaluate the condition.
2. If falsey, stop.
3. If truthy, execute the body.
4. Go back to step 1.

A loop must make progress or it runs forever:

```lox
while (true) {
  print "forever";
}
```

> [!NOTE]
> Base Lox has no `break` or `continue`. Those are possible extensions/challenges.

### 7.3 `for`

```lox
for (var i = 0; i < 3; i = i + 1) {
  print i;
}
```

| Clause      | Example     |
| ----------- | ----------- |
| Initializer | `var i = 0` |
| Condition   | `i < 3`     |
| Increment   | `i = i + 1` |

Execution order:

1. Run the initializer once.
2. Evaluate the condition.
3. If falsey, exit.
4. Execute the body.
5. Execute the increment.
6. Go back to step 2.

A `for` loop desugars to a `while`:

```lox
{
  var i = 0;
  while (i < 3) {
    print i;
    i = i + 1;
  }
}
```

The surrounding block matters: it gives the initializer's variable the right loop scope.

**Why no `for-in`/`foreach`?** It needs a common iteration protocol, which requires dynamic dispatch and iterator concepts not yet developed.

**Why no `do-while`?** Unnecessary for the book's goals; `while` teaches the same mechanics.

---

## 8. Functions

### 8.1 Calling a function

```lox
makeBreakfast("bacon", "eggs", "toast");
makeBreakfast();   // zero-arg calls still need parentheses
```

Without parentheses, the name refers to the function _value_; it does not invoke it.

### 8.2 Declaring a function

```lox
fun printSum(a, b) {
  print a + b;
}

printSum(10, 20);   // 30
```

The body is always a block.

### 8.3 Arguments vs. parameters

| Term          | Where       | Meaning                             |
| ------------- | ----------- | ----------------------------------- |
| **Argument**  | Call site   | The actual value passed             |
| **Parameter** | Declaration | The named variable that receives it |

```lox
fun greet(name) {          // name is a parameter
  print "Hello, " + name;
}

greet("Subh");             // "Subh" is an argument
```

A call has an _argument list_; a declaration has a _parameter list_. This vocabulary matters later for arity, call frames, and invocation.

### 8.4 Return values

```lox
fun add(a, b) {
  return a + b;
}

var result = add(4, 5);
print result;   // 9
```

Falling off the end of a function **implicitly returns `nil`**:

```lox
fun doSomething() {
  print "working";
}

var answer = doSomething();
print answer;   // nil
```

This is a language rule, not whatever the host language happens to return.

### 8.5 Functions are first-class values

A first-class function can be stored in a variable, passed as an argument, returned from another function, and used anywhere a value is accepted.

```lox
fun add(a, b) { return a + b; }

var operation = add;
print operation(2, 3);   // 5
```

Passing a function:

```lox
fun applyTwice(fn, value) { return fn(fn(value)); }
fun increment(n) { return n + 1; }

print applyTwice(increment, 10);   // 12
```

Returning a function:

```lox
fun getAdder() { return add; }

var anotherAdd = getAdder();
print anotherAdd(5, 6);   // 11
```

The book's composed-call example:

```lox
fun addPair(a, b) { return a + b; }
fun identity(a) { return a; }

print identity(addPair)(1, 2);   // 3
```

Read left to right:

1. Evaluate `addPair` to get a function value.
2. Pass it to `identity`.
3. `identity` returns the same function.
4. Call that function with `1` and `2`.

### 8.6 Nested functions

A function declaration is a statement, so functions can be declared inside functions:

```lox
fun outer() {
  fun inner() {
    print "I am inside outer";
  }

  inner();
}

outer();
```

This becomes powerful when `inner` uses variables from `outer`.

---

## 9. Closures

> One of the most important ideas in the chapter.

A **closure** is a function bundled with the surrounding variables it needs.

```lox
fun makeGreeter() {
  var greeting = "Hello";

  fun greet(name) {
    print greeting + ", " + name;
  }

  return greet;
}

var sayHello = makeGreeter();
sayHello("Subh");   // Hello, Subh
```

What happened:

1. `makeGreeter` starts executing.
2. It creates a local variable `greeting`.
3. It creates the nested function `greet`.
4. `greet` refers to `greeting`, which is neither its parameter nor its own local.
5. `makeGreeter` returns `greet`.
6. `makeGreeter` has finished executing.
7. `sayHello` still works and can read `greeting`.

**How can step 7 work?** The returned function keeps the surrounding environment it needs. That function-plus-environment combination is the closure.

```text
Closure = function code + captured lexical environment
```

(Conceptual — real implementations represent captured variables in different ways.)

**Why closures complicate memory management:** in a stack-only model, locals disappear when a function returns. A closure may still need one, so the runtime must preserve captured state as long as the closure can use it. Closures are not merely "functions inside functions" — the key property is that the nested function can **outlive** the activation that created it.

### Closures preserve mutable state

```lox
fun makeCounter() {
  var count = 0;

  fun next() {
    count = count + 1;
    return count;
  }

  return next;
}

var counter = makeCounter();
print counter();   // 1
print counter();   // 2
print counter();   // 3

var secondCounter = makeCounter();
print secondCounter();   // 1
```

Each call to `makeCounter` creates a distinct environment; the two closures do not share `count`.

### Implementation implications

- A function value must refer to its code.
- It must also reach the lexical environment it closes over.
- Captured variables must outlive the outer call when needed.
- The GC must account for closure/environment references.
- Name resolution must identify which lexical variable a reference points to.

The later resolving/binding and closure chapters are essential. A naive stack of locals is **not** enough.

> **Rust connection:** Rust closures capture their environment too, but ownership and lifetimes make those relationships explicit in the type system. Lox is dynamically typed and garbage-collected, so its runtime must preserve captured environments and reclaim them when unreachable.

---

## 10. Classes and Objects

Lox combines functional ideas (dynamic typing, lexical scope, closures) with object-oriented features. Objects bundle state and behavior: a class groups the methods that operate on its instances.

### 10.1 Classes vs. prototypes

| Class-based                               | Prototype-based                                          |
| ----------------------------------------- | -------------------------------------------------------- |
| A class describes methods and inheritance | Objects are the fundamental building blocks              |
| An instance stores per-object state       | Objects delegate behavior to other objects               |
| An instance refers to its class           | No separate class construct required                     |
| Method lookup: instance → class           | (JavaScript's foundation, despite modern `class` syntax) |

Lox chooses classes because programmers commonly want class-like organization, and users of prototype systems often rebuild classes on top anyway. This is the **waterbed** idea: complexity removed from the language reappears in user code or libraries.

### 10.2 Declaring a class

```lox
class Breakfast {
  cook() {
    print "Eggs are cooking!";
  }

  serve(who) {
    print "Enjoy your breakfast, " + who + ".";
  }
}
```

Methods look like function declarations without the `fun` keyword. Executing the declaration creates a class value bound to `Breakfast`.

Classes are first-class:

```lox
var type = Breakfast;
someFunction(Breakfast);
```

### 10.3 Creating an instance

Call the class like a function — there is **no `new` keyword**:

```lox
var breakfast = Breakfast();
```

```text
class value → allocate instance → run init (if present) → return instance
```

### 10.4 Fields

Fields are per-instance data. Assignment creates a field if it doesn't exist; fields need not be declared in the class.

```lox
breakfast.meat = "sausage";
breakfast.bread = "sourdough";
```

Instances are independent:

```lox
var first = Breakfast();
var second = Breakfast();

first.meat = "bacon";
second.meat = "mushrooms";
```

### 10.5 `this`

Inside a method, `this` refers to the receiving instance:

```lox
class Breakfast {
  serve(who) {
    print "Enjoy " + this.meat + " with " + this.bread + ", " + who;
  }
}

var meal = Breakfast();
meal.meat = "bacon";
meal.bread = "toast";
meal.serve("friend");
```

When `meal.serve(...)` runs, `this` is `meal`. It's a runtime binding: the method needs to know which instance received the call.

### 10.6 Initializers

A method named `init` runs automatically when an instance is created. Arguments to the class call are forwarded to it:

```lox
class Breakfast {
  init(meat, bread) {
    this.meat = meat;
    this.bread = bread;
  }

  serve(who) {
    print "Enjoy " + this.meat + " and " + this.bread + ", " + who;
  }
}

var meal = Breakfast("bacon", "toast");
meal.serve("reader");
```

The object starts life in a useful state instead of requiring the caller to assign every field.

### Vocabulary

| Term         | Meaning                                      |
| ------------ | -------------------------------------------- |
| **Class**    | A value describing behavior and construction |
| **Instance** | A particular object with its own state       |
| **Field**    | Instance data                                |
| **Method**   | Behavior associated with the class           |
| **`this`**   | The receiver                                 |
| **`init`**   | Initializes a newly created instance         |

---

## 11. Inheritance

Lox supports **single inheritance** using `<`:

```lox
class Brunch < Breakfast {
  drink() {
    print "How about a coffee?";
  }
}
```

`Brunch` is the subclass (derived); `Breakfast` is the superclass (base).

**Why `<`?** To avoid adding an `extends` keyword or reserving more punctuation. Ruby uses a similar convention.

### 11.1 Inherited methods

```lox
class Breakfast {
  serve(who) { print "Enjoy breakfast, " + who; }
}

class Brunch < Breakfast {
  drink() { print "Coffee time"; }
}

var meal = Brunch();
meal.serve("reader");   // inherited from Breakfast
meal.drink();           // defined by Brunch
```

### 11.2 Inherited initializer

`init` is inherited too. A subclass can define its own and call the superclass's:

```lox
class Breakfast {
  init(food) {
    this.food = food;
  }
}

class Brunch < Breakfast {
  init(food, drink) {
    super.init(food);
    this.drink = drink;
  }
}

var meal = Brunch("toast", "coffee");
```

`super.init(food)` invokes the superclass implementation of `init`, using the current instance as the receiver.

### 11.3 `super` vs. `this`

| Keyword | Meaning                                            |
| ------- | -------------------------------------------------- |
| `this`  | The current receiving instance                     |
| `super` | Access the superclass's implementation of a method |

When a subclass overrides a method, `super` lets it call the parent's version instead of re-selecting the override.

### 11.4 Single inheritance

One direct superclass per class — no multiple inheritance, interfaces, or mixins. A subclass inherits **methods**, not predeclared fields; since fields are created dynamically, each instance's field state is independent.

---

## 12. Primitives Are Not Full Objects

Lox has classes but is **not a pure object-oriented language**. In a pure OO model, every value — including numbers and Booleans — would be an object.

In Lox, primitives are not class instances and do not have user-defined fields or methods. This is an implementation compromise: primitives are introduced before classes, and making them full objects would complicate the implementation without serving the book's goals.

> Don't assume `123` behaves like a user-created instance.

---

## 13. The Standard Library

Intentionally tiny:

| Built-in  | Description                                                            |
| --------- | ---------------------------------------------------------------------- |
| `print`   | Statement that evaluates an expression and displays it                 |
| `clock()` | Returns seconds elapsed since the program started; used for benchmarks |

That's almost all. A practical language would need string utilities, collections, file I/O, input handling, math functions, networking, date/time, and error/process utilities.

**Why omit them?** The book is about language implementation; a large library would consume space without teaching interpreter and VM concepts.

---

## 14. Design Note: Expressions vs. Statements

An easy-to-skip but important design lesson. Lox has both:

```lox
print 1 + 2;   // statement
1 + 2          // expression
```

Some languages make nearly everything an expression — `if`, blocks, even declarations produce values:

```text
var result = if condition then 10 else 20
```

In Lox, `if` is a statement, so this is **not** valid:

```lox
var result = if (condition) 10 else 20;   // ❌
```

If everything is an expression, the designer must answer:

- What value does an `if` return? A block? A loop? A declaration?
- How do these constructs fit into operator precedence?
- Does a function implicitly return its final expression?
- What happens when statement-like constructs nest inside expressions?

| Benefits                                 | Costs                                       |
| ---------------------------------------- | ------------------------------------------- |
| Fewer distinctions between constructs    | More semantic decisions                     |
| More composability                       | More grammar interactions                   |
| Concise expression-oriented code         | Potentially surprising return values        |
| Implicit returns shorten small functions | More complexity around statement boundaries |

Lox chooses statements partly because C-like statement syntax is familiar and straightforward.

> **The broader lesson:** there is no universally correct answer. Language design is a set of trade-offs. A good designer makes rules coherent and predictable rather than adding features without considering their interactions.

---

## 15. Implementation Consequences to Remember

This chapter is a specification preview. Each rule becomes implementation work later.

| #   | Rule                                 | Consequence                                                                |
| --- | ------------------------------------ | -------------------------------------------------------------------------- |
| 1   | Four built-in types                  | Runtime must represent and distinguish Boolean, Number, String, Nil        |
| 2   | Numbers are doubles                  | No separate integer type; mind formatting and precision                    |
| 3   | Dynamic typing                       | Operations inspect runtime values; invalid combinations are runtime errors |
| 4   | `+` is overloaded                    | Number+Number adds, String+String concatenates, mixed is invalid           |
| 5   | Comparisons are numeric              | `<` `<=` `>` `>=` operate on numbers                                       |
| 6   | Equality is cross-type safe          | Different types are never equal; no host-language coercion                 |
| 7   | Truthiness is custom                 | Only `false` and `nil` are falsey; `0` and `""` are truthy                 |
| 8   | `and`/`or` short-circuit             | May skip the right operand; return an operand, not necessarily a Boolean   |
| 9   | Expressions ≠ statements             | Expressions produce values; statements produce effects                     |
| 10  | `print` is a statement               | Part of the grammar, not a built-in function                               |
| 11  | Uninitialized → `nil`                | Must be explicit runtime behavior                                          |
| 12  | Blocks introduce scope               | More than formatting braces                                                |
| 13  | Functions are first-class            | Stored, passed, returned, called through variables                         |
| 14  | No explicit return → `nil`           | Must be implemented consistently                                           |
| 15  | Closures capture variables           | Nested functions can outlive their creator                                 |
| 16  | Classes are first-class              | Storable in variables, passable to functions                               |
| 17  | Calling a class constructs           | No `new` keyword                                                           |
| 18  | Fields are dynamic                   | Assigning an unknown field creates it                                      |
| 19  | `this` is the receiver               | Methods use it to access the current instance                              |
| 20  | `init` initializes                   | Runs automatically during instance creation                                |
| 21  | Single inheritance                   | At most one superclass                                                     |
| 22  | `super` accesses superclass behavior | Not another name for `this`                                                |
| 23  | Primitives aren't instances          | Numbers and Booleans don't gain instance methods                           |
| 24  | Tiny standard library                | An educational language, not a production one                              |

---

## 16. Common Misunderstandings and Traps

| #   | Misconception                                        | Reality                                                                                   |
| --- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1   | "Dynamic typing means values have no types."         | Values have types; the runtime checks operations against them.                            |
| 2   | "Zero is false."                                     | Zero is truthy in Lox.                                                                    |
| 3   | "An empty string is false."                          | The empty string is truthy.                                                               |
| 4   | "`and`/`or` always return `true` or `false`."        | They return an operand per short-circuit rules.                                           |
| 5   | "The right side of `and`/`or` always runs."          | It may be skipped — side effects matter.                                                  |
| 6   | "Lox converts numbers to strings for `+`."           | Number + String is an error.                                                              |
| 7   | "`1` and `true` are equal because the host says so." | Different Lox types are never equal.                                                      |
| 8   | "An expression statement prints its value."          | It evaluates and discards the value.                                                      |
| 9   | "A function name automatically calls the function."  | Parentheses are required, even with zero args.                                            |
| 10  | "Locals always disappear when a function returns."   | A closure may keep captured variables alive.                                              |
| 11  | "Declaring a class constructs an instance."          | A class is a value; _calling_ it creates an instance.                                     |
| 12  | "Fields must be declared in the class."              | Assignment adds fields dynamically.                                                       |
| 13  | "`this` and `super` mean the same thing."            | `this` is the receiver; `super` selects superclass behavior.                              |
| 14  | "Lox has a large standard library."                  | It is intentionally minimal.                                                              |
| 15  | "The chapter specifies every edge case."             | It's informal; later chapters formalize details, and the challenges ask you to find gaps. |

---

## 17. Practical Examples: Predict the Output

Try predicting each output before expanding the answer.

### Example A — dynamic variable

```lox
var x = 10;
print x;
x = "ten";
print x;
```

<details><summary>Answer</summary>

```text
10
ten
```

A variable can hold values of different types over time.

</details>

### Example B — truthiness

```lox
print !0;
print !"";
print !nil;
print !false;
```

<details><summary>Answer</summary>

```text
false
false
true
true
```

Only `nil` and `false` are falsey.

</details>

### Example C — short-circuit

```lox
fun announce() {
  print "called";
  return true;
}

print false and announce();
print true or announce();
```

<details><summary>Answer</summary>

```text
false
true
```

`"called"` is never printed — both right-hand calls are skipped.

</details>

### Example D — operand-returning logic

```lox
print "primary" or "backup";
print nil or "backup";
print 0 and "continued";
```

<details><summary>Answer</summary>

```text
primary
backup
continued
```

A truthy left operand wins for `or`. A falsey left operand makes `or` evaluate and return the right. Zero is truthy, so `and` evaluates and returns its right operand.

</details>

### Example E — block scope

```lox
var message = "outer";

{
  var message = "inner";
  print message;
}

print message;
```

<details><summary>Answer</summary>

```text
inner
outer
```

The inner declaration shadows the outer variable within the block.

</details>

### Example F — implicit `nil` return

```lox
fun logMessage() {
  print "working";
}

var result = logMessage();
print result;
```

<details><summary>Answer</summary>

```text
working
nil
```

</details>

### Example G — closure state

```lox
fun makeCounter() {
  var n = 0;

  fun increment() {
    n = n + 1;
    return n;
  }

  return increment;
}

var c = makeCounter();
print c();
print c();
print c();
```

<details><summary>Answer</summary>

```text
1
2
3
```

The closure retains and mutates the captured variable `n`.

</details>

### Example H — class and instance

```lox
class Dog {
  init(name) {
    this.name = name;
  }

  speak() {
    print this.name + " says woof";
  }
}

var dog = Dog("Milo");
dog.speak();
```

<details><summary>Answer</summary>

```text
Milo says woof
```

Calling `Dog` constructs an instance; `init` sets its field; `speak` uses `this` to reach that instance.

</details>

### Example I — inheritance

```lox
class Animal {
  speak() { print "some sound"; }
}

class Dog < Animal {
  bark() { print "woof"; }
}

var d = Dog();
d.speak();
d.bark();
```

<details><summary>Answer</summary>

```text
some sound
woof
```

`Dog` inherits `speak` and defines `bark` itself.

</details>

---

## 18. Chapter Challenges

### Challenge 1 — Write sample Lox programs and run them

Don't only write happy-path examples. Probe the boundaries:

- [ ] Add two numbers
- [ ] Concatenate two strings
- [ ] Add a number and a string
- [ ] Compare values of different types
- [ ] Test zero and empty-string truthiness
- [ ] Test short-circuiting with a function that prints
- [ ] Declare a variable without an initializer
- [ ] Shadow a variable inside a block
- [ ] Return a function from another function
- [ ] Create two independent counters
- [ ] Add fields to two different instances
- [ ] Override a method in a subclass
- [ ] Call `super` from an overridden method

For every experiment, record:

1. The source code
2. The output or error
3. Why that behavior follows from the language rules
4. Whether the chapter explicitly specified it or left it open

### Challenge 2 — List what the informal introduction leaves unspecified

- What happens on division by zero?
- How are strings escaped?
- Are strings compared by contents?
- What happens when a variable is read before declaration?
- What happens when a variable is redeclared in the same scope?
- What is the maximum number of function arguments?
- What happens with the wrong number of arguments?
- What is the exact evaluation order of every expression?
- What if a class inherits from something that isn't a class?
- What happens if a method is missing?
- Can a class inherit from itself?
- What happens when a field and a method share a name?
- How are runtime errors reported and recovered from?

> Don't invent answers and assume the book promised them. Mark them as questions, then find the later chapter that defines them.

### Challenge 3 — What would make Lox annoying for real-world use?

Beyond the tiny standard library:

- Arrays/lists and maps/dictionaries
- More collection abstractions
- Modules and packages
- Better string and Unicode facilities
- Exceptions or structured error handling
- File and process APIs
- Iterators and `for-in` loops
- More numeric operators, including modulo
- A richer object model
- Visibility/encapsulation rules
- Interfaces or traits
- Concurrency and async
- Debugging and introspection

The goal isn't to add every feature — it's to understand that language design is constrained by implementation cost, usability, consistency, and purpose.

---

## 19. How to Study This Chapter

Don't rush into implementation. First make sure you can explain the language's behavior without the notes.

**Study loop**

1. **Read one concept** — e.g., dynamic typing.
2. **Close the notes** — explain it in your own words, as if teaching another programmer.
3. **Write a tiny Lox program** — use the feature concretely.
4. **Predict the output** before running it — reason about semantics, don't guess.
5. **Run it** in the reference interpreter if available and compare.
6. **Explain surprises** — find the exact rule responsible.
7. **Connect to implementation** — what must the scanner, parser, evaluator, environment, call frame, object representation, or GC do to support this?

**Two-column notebook**

| Language rule                 | Implementation consequence         |
| ----------------------------- | ---------------------------------- |
| Only `false`/`nil` are falsey | Dedicated truthiness function      |
| Number + Number adds          | Runtime type inspection            |
| String + String concatenates  | String allocation/concatenation    |
| Functions are first-class     | Function runtime value             |
| Closures capture variables    | Persistent lexical environment     |
| Fields are dynamic            | Runtime property storage           |
| Classes support inheritance   | Class metadata / superclass lookup |

This turns a language tour into an implementation roadmap.

---

## 20. Connections to Rust and C Systems Programming

### 20.1 Tagged values

A dynamically typed language needs values whose runtime types vary. A tagged union is the natural model:

```text
tag + payload
```

The tag tells the runtime how to interpret the payload. Rust enums make this explicit and safe; in C, the programmer must maintain the tag/payload invariant manually.

### 20.2 Runtime type checking

Evaluating `left + right` means asking: Number + Number? String + String? Anything else? — a small example of dynamic dispatch on runtime value categories.

### 20.3 Environments and lexical scope

An environment is a mapping plus a reference to its enclosing environment. Blocks and function calls introduce nested environments. This is not merely a hash-map problem — correct name resolution and lifetime behavior matter.

### 20.4 Closures and lifetime

Closures expose a deep runtime issue: a value can refer to data whose original stack frame has returned.

In a GC'd runtime, captured environments can be heap-allocated and retained while reachable. In Rust, design ownership carefully: `Rc`/`RefCell` can work for a single-threaded prototype; a different architecture may be needed for concurrency.

> [!TIP]
> Don't reach for `Arc`/`Mutex` automatically. Choose ownership and synchronization based on the runtime's actual threading model.

### 20.5 Object representation

| An instance needs                    | A class needs          |
| ------------------------------------ | ---------------------- |
| Its class                            | A name / identity      |
| Its fields                           | Its methods            |
| Possibly references to other objects | Its superclass, if any |

Method lookup follows the class/superclass chain. Field lookup and method lookup are related but distinct operations.

### 20.6 Garbage collection

Closures, classes, instances, and strings reference one another, forming object graphs. A tracing collector must know the roots and traverse references correctly. This is where language implementation meets systems engineering: memory layout, pointer representation, object graphs, allocation, lifetime, and runtime invariants.

---

## 21. Self-Test

Can you explain these without looking?

1. What is the difference between syntax and semantics?
2. What does dynamically typed mean?
3. Does dynamic typing mean values have no types?
4. What are Lox's four built-in types?
5. What numeric representation does Lox use?
6. Why is `"123"` not the same as `123`?
7. Which operands are valid for `+`?
8. Why is `1 == "1"` false?
9. What are the only two falsey values?
10. Why is zero truthy?
11. What does short-circuit evaluation mean?
12. Why can `and` and `or` return non-Boolean values?
13. What is the difference between an expression and a statement?
14. What does a block do besides grouping statements?
15. What value does an uninitialized variable contain?
16. What is the difference between an argument and a parameter?
17. What does a function return if it reaches its end?
18. What does first-class function mean?
19. What is a closure?
20. Why can closures complicate stack-based local-variable storage?
21. What is the difference between a class and an instance?
22. What happens when a class is called?
23. What does `init` do?
24. What does `this` refer to?
25. What does `super` do?
26. Does Lox support multiple inheritance?
27. Are numbers ordinary class instances?
28. Why is the standard library so small?
29. What trade-offs appear when every construct is an expression?
30. Which rules require custom runtime behavior rather than the host language's behavior?

If you can answer all 30 clearly and write small examples for each, you have a strong understanding of Chapter 3.

---

## 22. Final Mental Model

| Concept         | Summary                                                                                                   |
| --------------- | --------------------------------------------------------------------------------------------------------- |
| **Values**      | Boolean, Number, String, Nil                                                                              |
| **Expressions** | Arithmetic, comparison, equality, logic, grouping, calls                                                  |
| **Statements**  | `print`, expression statement, block, `var`, `if`, `while`, `for`, `fun`, `return`, `class`               |
| **State**       | Variables and lexical environments                                                                        |
| **Functions**   | First-class; parameters receive arguments; `return` sends a value back; falling off the end returns `nil` |
| **Closures**    | Function code plus the lexical environment it needs                                                       |
| **Objects**     | Classes create instances; instances hold dynamic fields; methods operate on the receiver via `this`       |
| **Inheritance** | Single superclass; inherited methods; `super` accesses superclass behavior                                |
| **Memory**      | Automatically managed; tracing GC becomes a major topic in clox                                           |

> **The deepest lesson of Chapter 3:** a programming language is a collection of precise behavioral promises. An interpreter must preserve those promises — including the surprising ones: zero is truthy, logical operators short-circuit and return operands, functions can outlive their creating calls, and class calls construct instances.

As you move on to scanning and parsing, keep returning to this chapter. The scanner and parser recognize the language's _structure_; the runtime gives that structure _meaning_. Chapter 3 tells you what meaning you are trying to implement.
