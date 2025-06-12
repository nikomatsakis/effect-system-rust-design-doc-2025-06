---
title: 'All that''s old is new again; or, a fresh const proposal.'

---

# All that's old is new again; or, a fresh const proposal.

> *inspired by [fee1-dead](https://hackmd.io/@beef/rJ1We7aCkl?edit) and very similar to Oli's original proposal*

## TL;DR

This section summarizes the key ideas for those who have been following. The remainder of the doc explores in a more conversation and in-depth style with the intention of giving a feel for how users can learn the ideas.

### MVP

```rust
(const) trait Example {
    const fn always_const();
    
    fn maybe_const();
}

impl<A> const Example for (A,)
where
    A: (const) Example,
{
    const fn always_const() {
        A::always_const(); // OK
        A::maybe_const(); // ERROR
    }
    
    fn maybe_const() {
        A::always_const(); // OK
        A::maybe_const(); // OK
    }
}

const fn example<T: (const) Example>() {}
```

Some anticipated questions that are covered in the FAQ:

* [Why move away from `(const)` on each method?][q1]
* [How do you cover all the "quadrants"?][q2]
* [What about other kinds of effects?][q3]

The short version of how I got here is this:

* Writing examples with `(const)` on every method felt weird and surprising. It felt a lot more intuitive to think of moving the `const` from each function up onto the trait and impl declaration as a kind of "default".
* Looking forward, we will want to be able to add additional "one-off" effects (e.g., `avx512`) atop a base set (e.g., the default effects, or `const`). That gives us a way to set the *floor* at the trait level but have individual methods opt-in to having more effects (i.e., not being const).
* The end result feels a lot lighterweight but seems to be more flexible.


## Walkthrough

We'll explain the concept by a series of examples. Along the way, we'll desugar to 'associated effects' as a means of clarifying semantics of the notation, but we are not proposing to add associated effects or explicit effect notation to Rust (though it is a future possibility, one of many).

### Existing traits and fns

We start with existing notation. Nothing to see here, but we'll be using these example traits and functions throughout the piece, so give them a quick read.

```rust
trait Create {
    fn create() -> Self;
}

impl Create for i32 {
    fn create() -> Self {
        0
    }
}

impl<A: Create> Create for (A,) {
    fn create() -> Self {
        (A::create(),)
    }
}

fn make<A: Create>() -> A {
    A::create()
}
```

### Associated effect desugaring

Functions in Rust do not just produce a value, they can have various side-effects -- e.g., allocating memory, panicking, infinite loops, reading globals, syscalls, nondeterminism. We call this set of effects the "default" set. Many of the default side-effects that functions can have are not suitable for execution at compilation time (e.g., nondeterminism or syscalls). Users can declare a `const` fn which indicates that it has a more limited set of side-effects:

```rust
const fn make_i32() -> i32 {
    0
}
```

### The problem from generics

Generics pose a challenge for const functions:

```rust
const fn make<A: Create>() -> A {
    A::create() // ERROR
}
```

Although `make::<A>` would be safe to execute at compilation time for many types (e.g., `i32` or `(i32,)`), we can't be sure it's safe for *all* types, since some of them may have side effects. What we need is some way to say that trait methods will be const. There are a number of ways we can do this. We will walk through the options here and explain when you want to use each approach.

### Declaring const methods in a trait

The simplest option is to declare a method to be `const` as part of the trait:

```rust
trait Create {
    const fn make() -> Self;
}
```

When you do this, you are saying that *all* impls of the trait have to provide a `const` fn for `make`. So we would have to adjust our impls. This is no problem for `i32` and `(A,)`

```rust
// Variant:
//
// trait Create { const fn create() -> Self; }

impl Create for i32 {
    const fn create() -> Self {
    // -- had to add this keyword...
        0
    }
}

impl<A: Create> Create for (A,) {
    const fn create() -> Self {
    // -- ...and this one.
        (A::create(),)
    }
}
```

But what if we wanted to have a `Create` impl which *did* have side-effects? For example, here is a type `ErrorReported` whose existence implies that an error message was written out on stderr:

```rust
// Variant:
//
// trait Create { const fn create() -> Self; }

pub struct ErrorReported(());
impl Create for ErrorReported {
    const fn create() -> Self {
        eprintln!("Error occurred");
        // ----- ERROR: can't use `eprintln!` in const fn
        ErrorReported(())
    }
}
```

We can't use `eprintln!` in a const fn, so this won't work. That's too bad.

### Declaring "maybe const" traits

When you declare a const method in a trait, you are forcing all impls to supply a const fn. If you want to give impls the *choice* as to whether their methods can have side-effects or not, you can declare the trait itself to be `(const)`. The parentheses here can be read as "maybe" -- as in, the trait methods *may* or *may not* be const:

```rust
(const) trait Create {
    fn create() -> Self;
}
```

The impl can then declare which version it wants by implementing either `Create` (regular functions) or `const Create` (const functions):

```rust
impl const Create for i32 {
    fn create() -> Self {
        // You don't need to declare the fn as const,
        // it is inherited from the trait.
        0
    }
}

pub struct ErrorReported(());
impl Create for ErrorReported {
    fn create() -> Self {
        eprintln!("Error occurred"); // OK!
        ErrorReported(())
    }
}
```

### Conditionally const bounds

But remember that we have seen 3 impls of `Create` in total: `i32`, `ErrorReported`, and one for all 1-tuples `(A,)`. This last impl is interesting: should it be declared as *const* or not? The answer of course depends on the type `A`: `(i32,)` ought to implement `const Create` but `(ErrorReported,)` should not. We can express this by using a "maybe const" bound:

```rust
impl<A> const Create for (A,)
where
    A: (const) Create, // <-- "maybe const"
{
    fn create() -> Self {
        (A::create(),)
    }
}
```

In fact, we can use this same notation to write our original `make` function:

```rust
const fn make<A: (const) Create>() -> A {
    A::create()
}
```

In both cases we see the same pattern: the function/trait is declared as `const` but it has "maybe const" bounds. How should we understand this? Does it have side-effects or not?

Declaring the **function/trait as const** says that it does not have side-effects within its immediate body. For example, the function `make` does not have any side-effects, and is therefore const-safe. This first part explains why, for example, this function would not compile:

```rust
const fn loud_make<A: (const) Create>() -> A {
    eprintln!(
        "Making a value of type {}",
        type_name::<A>(),
    ); // ERROR: eprintln! not allowed in const fn
    A::create()
}
```

However, it can still have *indirect* side-effects because of the call to `A::create()`. Therefore if you call `make::<ErrorReported>`, it will have side-effects.

### Always vs conditionally const bounds

Compare these two functions, which look very similar:

```rust
const fn make1<A: const Create>() -> A { ... }
const fn make2<A: (const) Create>() -> A { ... }
```

What is the difference? By declaring `const Create`, `make1` indicates that it is needs a const impl. This also implies that `make1` cannot have side-effects: the direct function body must be const, and it doesn't have any `(const)` bounds.

`make2`, in contrast, is saying that it can take any impl. If it is given a type with a `const Create` impl, like `i32`, it will be const-safe (no side-effects). But if it is given a type that just implements `Create`, like `ErrorReported`, it may well have side-effects.

You usually want `make2`, as it is more flexible. But there are times where `make1` would be useful. For example, consider this definition:

```rust
const fn make1<A: const Create>() -> &'static A {
    &const { A::create() }
}
```

Here, the call to `A::create` occurs inside of a const-expression, i.e., at compilation time. This is used to create a `'static` reference. This means that `A` *must* be executable at compilation time. You can't write a function like this with a `(const) Create` bound.

### Combining conditionally const traits and const methods

Most of the time you either want *all* the methods in the trait to be const or *none* of them. But there are times when some methods *must* be const but others only *may* be. Consider an expanded version of `Create`:

```rust
(const) trait Create {
    const fn tag(offset: u32) -> u32;
    
    fn create() -> Self;
}
```

In this version, every `Create` type also has the ability to create a unique "type tag". This has to be computable at compilation time and hence *cannot* have side-effects. But it also has the ability to create the value, which may have side-effects.

When types implement this trait, they must declare the tag function as `const`, independent of whether they are implementing `Create` or `const Create`:

```rust
impl const Create for i32 {
    const fn tag(offset: u32) -> u32 {
        0 // all integers have tag 0
    }
    
    fn create() -> Self {
        0
    }
}

impl<A> const Create for ErrorReported
where
    A: (const) Create,
{
    const fn tag(offset: u32) -> u32 {
        A::tag(offset + 1)
    }
    
    fn create() -> Self {
        0
    }
}

impl Create for ErrorReported {
    const fn tag(offset: u32) -> u32 {
        offset + 1
    }
    
    fn create() -> Self {
        0
    }
}
```

### Const methods and `(const)` bounds

Let's take a closer look at this impl:

```rust
impl<A: (const) Create> const Create for (A,) {
    const fn tag(offset: u32) -> u32 {...}
    fn create() -> Self {...}
}
```

The rule generally is that a `const` fn cannot have direct side-effects but it *can*, via `(const)` bounds, invoke methods which may have side-effects. So would `tag` be allowed to call `A::create`, since the where-clause `A: (const) Create` is in scope?

```rust
impl<A: (const) Create> const Create for (A,) {
    const fn tag(offset: u32) -> u32 {...}
        A::create() // ERROR
    }
    fn create() -> Self {...}
}
```

As the examples indicates, it is not. The reason is that methods in an impl have to obey what is declared in the *trait*. And the *trait* version of `tag` doesn't have any `(const)` bounds in scope. Therefore, it must not have any side-effects at all.

You could declare a trait with `()` bounds:

```rust
trait Foo<T: (const) Create> {
    const fn foo() -> T;
}
```

Here, the `const` method follows the usual rules: it itself cannot have side-effects, but it can invoke `T::create()`:

```rust
impl<A> Foo<A> for ()
where
    A: (const) Create,
{
    const fn foo() -> T {
        A::create() // OK
    }
}
```

This means that calls to `foo` are only considered `const` if the type parameter has a `const Create` impl:

```rust
const fn good<A, B>()
where
    A: Foo<B>,
    B: const Create,
{
    A::foo() // OK: `foo` is known to be const
}

const fn also_good<A, B>()
where
    A: Foo<B>,
    B: (const) Create,
{
    A::foo() // OK: `foo` is known to be const
             // modulo `B::create`, same as us
}

const fn not_good<A, B>()
where
    A: Foo<B>,
    B: Create,
{
    A::foo() // ERROR: `foo` is not known to be const
}
```

## Frequently asked questions and future extensions

### Can you translate this to a more formal model?

Yes, let's map these rules to associated effects as modeled in a-mir-formality. To start we formalize the notion of a *side-effect*. The full set of effects isn't important, but we need at least two aliases (`const` and `default` where `const <= default`). Let's use this set for now:

```
Effect =
  | nondet  // not strictly deterministic
  | loop    // infinite loop
  | const   // = nondet, loop
  | syscall // syscall like eprintln
  | panic   // panic
  | alloc   // memory allocation
  | static  // access to a static/global variable
  | default // = const, syscall, panic, alloc, static
  | ...     // ...others we will consider later
  | Effect, Effect             // union
  | <T0 as Trait<...>>::Effect // associated effect
```

Traits are extended with the ability to declare associated effects. Regular traits do not have an associated effect. `(const)` traits have exactly one, named `Effect`:

```rust
// trait RegularExample { ... }
trait RegularExample { /* ... */ }

// (const) trait MaybeConstExample { ... }
trait MaybeConstExample {
    do Effect;
    
    // ...
}
```

Functions and methods are annotated with an explicit effect written with `#[do(...)]` notation. `const` functions get the effect `const` and regular functions get the effect `default`:

```rust
// const fn foo()
#[do(const)]
fn foo() {}

// fn bar()
#[do(default)]
fn bar() {}
```

The full set of effects for a `fn` depends on how/where it is declared. It will be the union of:

* For a free function:
    * The function's declared effect (`const`, `default`) plus the associated effect from any `(const)` bounds.
* For a method in a trait declaration:
    * If the function has a declared effect (e.g., `const fn`), use that.
    * Otherwise, when the function has no declared effect, the effect depends on the trait declaration:
        * For a `(const)` trait, include `const` and the trait associated effect.
        * For a regular trait, include `default`.
    * Always include any in-scope `(const)` bounds on the trait or the function.
* For a method in an impl:
    * Effects as declared in the trait.

Therefore:

```rust
// (const) trait Create {
//   const fn tag(i: u32) -> u32;
//   fn create() -> Self;
// }
trait Create {
    do Effect;
    
    #[do(const)]
    fn tag() -> u32;
    
    #[do(const, Self::Effect)]
    fn create() -> Self;
}
```

### Would `const trait` have any meaning?

I think `const trait` has a natural meaning which is to declare all functions within as `const fn` -- annotating a trait is best thought of as setting a default for all the unadorned functions within. However, this doesn't seem very useful, and it raises some questions (e.g., do users have to write `T: const Foo`?). I've therefore left it out as an option.

[q3]: #Extending-to-more-kinds-of-effects

### Extending to more kinds of effects

What about other kinds of effects? We can extend the system readily. The idea would be to allow a `do` declaration with an explicit list of effects. I don't know what would be the best syntax, but it's clear that finding the best one will take some work. Here is a strawperson proposal but please don't dwell on it too much.

To start, instead of just `const`, functions can be annotated with a customized list of effects. This is introduced with the `do(effect1, effect2)` keyword -- so a `const` fn just has the base set of effects, but a `const do(...)` function adds additional `...` effects. e.g., `prints` is declared as doing `const` things but also performing syscalls:

```rust
const do(syscall) fn prints() {
    eprintln!("Hi.");
}
```

You could also leave off the `const`, in which case the set of effects includes all the default effects. Writing `do(syscall)` here is pretty pointless, because `syscall` is part of the default set, so this declaration has no effect:

```rust
do(syscall) fn prints() {
// -------- lint: `do(syscall)` is redundant
    eprintln!("Hi.");
}
```

However, we'll see later that we can use this with things like `do(avx512)` that are not part of the default effect set.

We can permit the declaration of named aliases for commonly used sets with `do x = y`. These can also begin with `const` to indicate the floor is the `const` set of effects:

```rust
const do cs = syscall;

cs fn prints() {
    eprintln!("Hi.");
}
```

Traits declared as `(const)` (or other sets of effects) can naturally be extended to permit more effects as well. So for `ReportError` we could implement `cs Create` which is more precise that `Create` since it excludes things like `panic`:

```rust
struct ReportError(())
impl cs Create for ReportError {
    fn create() -> Self {
        eprintln!("Error occurred"); // OK!
        ErrorReported(())
    }
}
```

This would be translated to the "associated type formalism" like so:

```rust
struct ReportError(())
impl Create for ReportError {
    do Effect = const, syscall;
    
    #[do(Self::Effect)]
    fn create() -> Self {
        eprintln!("Error occurred");
        ErrorReported(())
    }
}
```

### What about modeling target features as effects?

Allowing for more sets of effects means we can start to track effects beyond the default set. One possible example is the use of *target features*, meaning codegen that uses advanced instructions. For example, there might be a `avx512` target feature for the ability to use the AVX512 instructions offered by some Intel processors.

```rust
do(avx512) fn compute() {
    ...
}
```

You could then have implementations that declare the effects they need:

```rust
impl do(avx512) Create for MyType {
    fn foo() {
        // has the `do(avx512)` effect in scope,
        // which in turn means better codegen
    }
}
```

One could write a generic function that *may* have additional effects beyond the default like so:

```rust
fn create<T>()
where
    T: (const) Create,
    // ------- indicates that  
{
    
}
```

### What about async?

Async could work in ~the same way as `avx512` and friends. The thing is that when a function *may* have the async effect, the compiler needs to compile it as an async function -- which implies that we have to monomorphize fns many times based on the effects that show up in their where-clauses. All "marker" effects (which do not affect ABI) can be compiled as one function, but things like async, try, or target features are different.

### What if I want a trait that has some methods always const, some condition, and others never const?

If we permit the extended `do` notation, you can do the "Everything everywhere all at once" case fairly easily. This trait defines *computations*, where:

* some computations be done at compilation time and others not;
* but *all* computations have the ability to do a compilation time *estimate* of the result;
* and *all* computations can log their results at runtime.

```rust
(const) trait Computation {
    const fn estimate(input: u32) -> u32;
    
    fn compute(input: u32) -> u32;
    
    // Explicitly add back the default effects:
    do(default) fn log();
}
```

Per the rules given above, this would correspond to:

```rust
trait Computation {
    do Effect;
    
    #[do(const)]
    fn estimate(input: u32) -> u32;
    
    #[do(Self::Effect)]
    fn compute(input: u32) -> u32;
    
    #[do(default)]
    fn log();
}
```

[q1]: #Why-move-away-from-const-fn-methods

### Why move away from `(const) fn` methods?

For a while I was advocating for traits where each "maybe const" method was annotated with `(const)` fn. However, when returning to those examples with a fresh eye, it seemed clear that the visual clutter and overhead was going to be a problem:

```rust
impl<A> const Default for (A,)
where
    A: (const) Default
{
    (const) fn default(self) -> Self {
        // how many times do I have to write `(const)`
    }
}
```

Moreover, it created some inconsistencies. For example, a "maybe const" free function looks like...

```rust
const fn maybe_const<A: (const) Default>() {}
```

...but that same function in a trait was likely to be written with `(const)`. Writing `const fn` in the impl actually implied a rather different meaning, a refinement on the trait.

Under the new proposal, while there is still a discontinuity, things "feel different" to me. The `const` from the free function just *moves*, to the trait name, which indicates that it applies to all the functions within the trait. This feels like it can be explained more readily to me.

[q2]: #But-what-about-the-quadrants

### But what about the quadrants?

The other motivation was enabling all the "quadrants" -- that is, being able to express ideas like "always maybe const", "never const", and so forth. However, what I realized is that we could recover that capability by "layering on" extra effects atop the default. So if you have a `(const) trait`, this implies that `fn` are "maybe `const`" by default, but we can recover the idea of a function being "never const" by just *adding* the effect to the function (e.g., `do(default)` or some such). Moreover, this idea of *adding* effects is needed anyway in order to model extended effects like target features. In that context, working through examples made it clear that we would want to be able to take a "base set" of effects and then "add" a few extras. For example:

```rust
do(avx512) fn foo() {}
```

While I'm unclear about the *precise syntax* here (e.g., maybe `#[do(avx512)]` would be better? Or put the `do` somewhere else?), it does seem clear that we are going to want to be able to layer on the extra effects atop a base.

To see what I mean, let's walk through all the options.

**Always const.**

```rust
trait Foo {
    const fn foo();    
}

// becomes

trait Foo {
    #[do(const)]
    fn foo();    
}
```

**"Always maybe" const based on where-clauses.**

```rust
trait Foo {
    const fn foo<T>()
    where
        T: (const) Default;
}

trait Bar<T: (const) Default> {
    const fn foo();
}

// become

trait Foo {
    #[do(const, <T as Default>::Effect)]
    fn foo<T: Default>();    
}

trait Bar<T: Default> {
    #[do(const, <T as Default>::Effect)]
    fn foo();    
}
```

**"Maybe const" based on impls.**

```rust
(const) trait Foo<T: (const) Default> {
    // always const
    const fn bar();
    
    // const if
    // * `T: const Default`
    // * `U: const Default`
    const fn bar<U: (const) Default>();
    
    // const if:
    // * impl is for `const Foo`
    // * `T: const Default`
    fn foo();
    
    // const if:
    // * impl is for `const Foo`
    // * `T: const Default`
    // * `U: const Default`
    fn foo<U: (const) Default>();
}
```

**Never const.**

```rust
(const) trait Foo {
    // const if:
    // * impl is for `const Foo`
    fn maybe();

    // never const, no matter the impl
    do(default) fn never();
}
```

### What can't be expressed?

There are two things you cannot do in this system. Doing these really requires some more expressive syntax, likely a first-class notion of associated effects or something RTN-like. I would prefer not to add that until we understand better the use-cases.

* pick and choose which `(const)` bounds you want

```rust
trait Foo<T: Bar>  {
    do Effect;
    
    // ignores the `T: (const) Bar`
    #[do(Self::Effect)]
    const fn m1();
    
    // uses the `T: (const) Bar`
    #[do(Self::Effect, T::Effect)]
    const fn m2();
}
```

* have impls pick and choose which methods to make const.

```rust
trait Foo<T: Bar>  {
    do Effect1;
    do Effect2;
    
    #[do(Self::Effect1)]
    const fn m1();
    
    #[do(Self::Effect2)]
    const fn m2();
}
```

