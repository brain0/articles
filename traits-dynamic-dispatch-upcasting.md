# Traits, dynamic dispatch and upcasting

I recently hit a limitation of Rust when working with trait objects. I had a function that returned a trait object and I needed a trait object for one of its supertraits.

```rust
trait Super {}

trait Sub: Super {}

fn upcast(obj: Arc<dyn Sub>) -> Arc<dyn Super> {
    obj
}
```

To my surprise, the code did not compile:

```text
error[E0308]: mismatched types
 --> src/lib.rs:8:5
  |
7 | fn upcast(obj: Arc<dyn Sub>) -> Arc<dyn Super> {
  |                                 -------------- expected `std::sync::Arc<(dyn Super + 'static)>` because of return type
8 |     obj
  |     ^^^ expected trait `Super`, found trait `Sub`
  |
  = note: expected struct `std::sync::Arc<(dyn Super + 'static)>`
             found struct `std::sync::Arc<(dyn Sub + 'static)>`
```

So, I looked at [the reference](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions) which states:

> The following coercions are built-ins and, if `T` can be coerced to `U` with one of them, then an implementation of `Unsize<U>` for `T` will be provided:
>
> * `[T; n]` to `[T]`.
> 
> * **`T` to `U`, when `U` is a trait object type and either `T` implements `U` or
  `T` is a trait object for a subtrait of `U`.**
> 
> * `Foo<..., T, ...>` to `Foo<..., U, ...>`, when:
>     * `Foo` is a struct.
>     * `T` implements `Unsize<U>`.
>     * The last field of `Foo` has a type involving `T`.
>     * If that field has type `Bar<T>`, then `Bar<T>` implements `Unsized<Bar<U>>`.
>     * T is not part of the type of any other fields.

So, let's look at this: `dyn Super` is a trait object type and `dyn Sub` is a trait object type for one of its subtraits, and it does not work. Okay, let's try the other half of the sentence.

```rust
trait Super {}

fn to_trait_object<'a, T: Super + 'a>(t: Arc<T>) -> Arc<dyn Super + 'a> {
    t
}
```

This compiled just fine, but this doesn't:

```rust
trait Super {}

fn to_trait_object<'a, T: Super + ?Sized + 'a>(t: Arc<T>) -> Arc<dyn Super + 'a> {
    t
}
```

The compiler complains:

```text
error[E0277]: the size for values of type `T` cannot be known at compilation time
 --> src/lib.rs:6:5
  |
5 | fn to_trait_object<'a, T: Super + ?Sized + 'a>(t: Arc<T>) -> Arc<dyn Super + 'a> {
  |                        - this type parameter needs to be `std::marker::Sized`
6 |     t
  |     ^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `T`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: required for the cast to the object type `dyn Super`
```

So, the reference is clearly misleading here. I started out to explore why this doesn't work and if I can do something about it.

Let's start with some basics.

## What is dynamic dispatch?

The most common and idomatic way to use traits in Rust is through generics:

```rust
trait TypeDescription {
    fn get_description(&self) -> String;
}

impl TypeDescription for u8 {
    fn get_description(&self) -> String {
        format!("{} is an unsigned 8 bit integer.", self)
    }
}

impl TypeDescription for i64 {
    fn get_description(&self) -> String {
        format!("{} is a signed 64 bit integer.", self)
    }
}

fn print_description<T: TypeDescription>(t: &T) {
    println!("{}", t.get_description());
}

fn main() {
    print_description(&42u8);
    print_description(&42i64);
}
```

The output of this program is as follows:

```text
42 is an unsigned 8 bit integer.
42 is a signed 64 bit integer.
```

So what actually happens here? The signature `fn print_description<T: TypeDescription>(t: &T)` defines a *generic function*. Every time the function is called, the compiler determines the type `T` of the first argument, checks if `T` implements the `TypeDescription` trait and if so, generates code for this function specific to the type arguments. Depending on the precise size, alignment and layout of the type, and the specifics of the trait implementation, the generated code might differ considerably. This process is often called *monomorphisation*.

Because the code for the function call is generated statically at compile time, this method of calling generic code is sometimes called *static dispatch*. If you think that this is all very similar to C++ templates, you are right: A similar monomorphisation process happens when C++ templates are instantiated.

Static dispatch is considered efficient, and it is a major reason why Rust performs so well in the presence of generic code: You don't pay an additional runtime cost compared to writing multiple almost identical functions for different types.

However, in order for static dispatch to work, the compiler must know all types at compile time. But that is not always possible. A common example is having a collection of items of different type that all implement a common trait. In Rust, this can be achieved with *trait objects*.

```rust
fn print_descriptions(ts: &Vec<Box<dyn TypeDescription>>) {
    for t in ts {
        println!("{}", t.get_description());
    }
}

fn main() {
    let ts: Vec<Box<dyn TypeDescription>> = vec![Box::new(42u8), Box::new(42i64)];

    print_descriptions(&ts);
}
```

The output of this program is the same as above. But how does this work, and how is it different from the first program? To understand this, we need to talk about *dynamically sized types*.

## Dynamically sized types

A dynamically sized type (or *DST*, sometimes also referred to as an *unsized type*) is a type whose size is unknown at compile time. But how can such a type even exist? After all, the compiler creates the values so it must know its size, right?

Let's look at the previous code example: We create a few values of different types, then put them in `Box`es and out those into a `Vec`. From now on, all we know about these types is their address and ... something else.

The first rule of DSTs is that they can only exist behind some kind of pointer, since their size is not known at compile time. But what is a pointer anyway? This is a non-comprehensive list of pointer types in Rust:

* `*const T`, `*mut T`
* `&T`, `&mut T`
* `Box<T>`
* `Rc<T>`
* `Arc<T>`
* `Pin<P>` where `P` is any of the above.

What all these types have in common is that their in-memory representation is a simple pointer, i.e. an integer the size of a machine word that refers to a memory address. The only difference between them is what the compiler allows you to do with them and what code it generates for them.

For the purpose of this section, it is sufficient to consider two types of DSTs: Slices and trait objects. A DST is created by a process that is sometimes called *unsizing*:

* First you create a pointer (see above) to a value of a sized type.
* This pointer is then coerced into the corresponding trait object type, which is a tuple of two values: the original pointer and *something else*. This coercion happens implicitly.

What's important about unsizing coercions is that the information needed to generate the second value must be known at compile time.

### Arrays coerce to slices

Arrays implicitly coerce to slices. The second value is simply the length of the slice.

Slices can also be created manually from a pointer and a length, but this is not a coercion, so in this case, the length need not be known at compile time. The standard library does this, for example in the `Deref` implementation of `Vec`.

While slices are definitely interesting, we won't discuss them further in this article.

### Sized types coerce to trait objects

Values of sized types implicitly coerce to trait objects for any *object-safe trait* they implement. The second value is a pointer to the so-called *vtable*. The vtable has many more names, Wikipedia says the following:

> A virtual method table (VMT), virtual function table, virtual call table, dispatch table, vtable, or vftable is a mechanism used in a programming language to support dynamic dispatch (or run-time method binding).

We'll explore the vtable in more detail.

## Trait objects and the vtable

The vtable is what allows Rust to call trait methods on a value without knowing its type. The vtable is generated at compile time and stored as part of the binary. As of Rust 1.43, the layout of the vtable is as follows (although rustc makes no guarantess about it):

| Field | Type |
| --- | --- |
| `drop_in_place` implementation | Pointer|
| size of the value | usize |
| minimum alignment of the value | usize |
| first trait function | Pointer |
| ... | ... |
| n'th trait function | Pointer |

Let's go through these items:

### `drop_in_place`

When a `Box` is dropped or the strong count of an `Rc` or `Arc` drops to zero, the standard library calls [`drop_in_place`](https://doc.rust-lang.org/stable/std/ptr/fn.drop_in_place.html) on the value it points to. For sized types, the compiler statically knows how to drop a value. For slices, it calls `drop_in_place` for every element. For trait objects, it calls the `drop_in_place` implementation that from the vtable.

### Size and alignment

The size and alignment are used to implement [`std::mem::size_of_val`](https://doc.rust-lang.org/stable/std/mem/fn.size_of_val.html) and [`std::mem::align_of_val`](https://doc.rust-lang.org/stable/std/mem/fn.align_of_val.html). They are also used during code generation in the internals of the compiler.

Since the size of the trait object is part of the vtable, logic dictates that **you cannot create a trait object from a DST (e.g. a slice)**.

### Pointers to the trait functions

To dynamically dispatch method calls, rustc needs function pointers to all trait methods (including supertraits). The order in which they appear in the vtable is unspecified.

This brings us to *object safety*: In order to create a vtable, the compiler needs to create a function pointer for all trait methods and the first argument must always be a pointer to the object itself. [Object safety](https://doc.rust-lang.org/reference/items/traits.html#object-safety) makes sure that this is always possible. In particular, you cannot dynamically dispatch generic methods.

## Upcasting

Let's come back to the original problem. Coming from object-oriented languages, upcasting is taken for granted. Imagine the following C++ code:

```c++
class Super {};

class Sub : public Super {};

void func_taking_super(Super& obj) {
    // ...
}

void func_taking_sub(Sub& obj) {
    func_taking_super(obj);
}
```

Or the following C# code:

```c#
class Super {}

class Sub {}

static class Methods {
    void FuncTakingSuper(Super obj) {
        // ...
    }

    void FuncTakingSub(Sub obj) {
        FuncTakingSuper(obj);
    }
}
```

In Rust, as we've seen in the beginning, this isn't always possible:

```rust
trait Super {}

trait Sub: Super {}

fn func_taking_super<T: Super + ?Sized>(obj: &T) {
    // ...
}

fn func_taking_super_dyn(obj: &dyn Super) {
    // ...
}

fn func_taking_sub(obj: &dyn Sub) {
    func_taking_super(obj);
    //func_taking_super_dyn(obj);
}
```

This compiles and works. But uncommenting the second line in `func_taking_sub` leads to the following compiler error:

```text
error[E0308]: mismatched types
  --> src/lib.rs:18:27
   |
18 |     func_taking_super_dyn(obj);
   |                           ^^^ expected trait `Super`, found trait `Sub`
   |
   = note: expected reference `&dyn Super`
              found reference `&dyn Sub`
```

But why doesn't it work? It is completely reasonable to expect that it should. After all, the compiler knows how to call any method of the trait `Super` on values of type `&dyn Sub`.

The problem is that the vtable has to be generated at compile time and the compiler does not know the actual type of `obj` when compiling the function `func_taking_sub`.

I can see two solutions to this:

### Solution 1

Change the layout of the vtable so that the vtables of supertraits are sub-tables of the main vtable. As far as I know, this is what C++ compilers do. For the following traits ...

```rust
trait Super { /* ... */ }
trait Sub: Super { /* ... */ }
```

... a vtable of `Sub` would look like this:

| Field | Type |
| --- | --- |
| `drop_in_place` implementation | Pointer|
| size of the value | usize |
| minimum alignment of the value | usize |
| first trait function of `Super` | Pointer |
| ... | ... |
| n'th trait function of `Super` | Pointer |
| first trait function of `Sub` | Pointer |
| ... | ... |
| m'th trait function of `Sub` | Pointer |

Then you can use the same vtable pointer when upcasting `Sub` to `Super`.

The problem here is that as soon as any trait in the chain has more than one supertrait, you'd have to repeat the `drop_in_place` pointer, the size and the alignment multiple times to allow upcasting to all possible supertraits. Consider the following traits ...

```rust
trait Super1 { /* ... */ }
trait Super2 { /* ... */ }
trait Sub: Super1 + Super2 { /* ... */ }
```

| | Field | Type |
| ---: | --- | --- |
| *vtable of `Sub` and `Super1` ->* | `drop_in_place` implementation | Pointer|
| | size of the value | usize |
| | minimum alignment of the value | usize |
| | first trait function of `Super1` | Pointer |
| | ... | ... |
| | m'th trait function of `Super1` | Pointer |
| *vtable of `Super2` ->* | `drop_in_place` implementation | Pointer|
| | size of the value | usize |
| | minimum alignment of the value | usize |
| | first trait function of `Super2` | Pointer |
| | ... | ... |
| | n'th trait function of `Super2` | Pointer |
| | first trait function of `Sub` | Pointer |
| | ... | ... |
| | o'th trait function of `Sub` | Pointer |

This way, the compiler would still be able to determine a vtable of both `Super1` and `Super2` from a vtable of `Sub`. However, this gets pretty complex when more traits are involved.
   
This is also part of why C++ class inheritance is so complex and I can understand why rustc developers would not want this complexity.

### Solution 2

When creating a vtable, generate the vtables of all possible supertraits, and include pointers to those supertrait vtables in the vtable itself.

| Field | Type |
| --- | --- |
| `drop_in_place` implementation | Pointer|
| size of the value | usize |
| minimum alignment of the value | usize |
| vtable of `Super1` | Pointer |
| vtable of `Super2` | Pointer
| first trait function of `Super1` | Pointer |
| ... | ... |
| m'th trait function of `Super1` | Pointer |
| `drop_in_place` implementation | Pointer|
| size of the value | usize |
| minimum alignment of the value | usize |
| first trait function of `Super2` | Pointer |
| ... | ... |
| n'th trait function of `Super2` | Pointer |
| first trait function of `Sub` | Pointer |
| ... | ... |
| o'th trait function of `Sub` | Pointer |

This is not nearly as complex and would (probably unnecessarily) increase binary size.

You could also combine these solutions and choose solution 1. where easily possible, but fall back to solution 2. otherwise. Both of these solutions add complexity to the compiler that may be undesirable.

## A practical solution

Due to the added complexity, I am unsure if Rust will ever allow upcasting trait objects. After all, I seem to be the first one to have cared about this.

However, there is a neat trick to solve this problem, at least for traits that you define yourself.

```rust
trait Super: AsDynSuper {}

trait AsDynSuper {
    fn as_dyn_super<'a>(self: Arc<Self>) -> Arc<dyn Super + 'a>
    where
        Self: 'a;
}

impl<T: Super + Sized> AsDynSuper for T {
    fn as_dyn_super<'a>(self: Arc<Self>) -> Arc<dyn Super + 'a>
    where
        Self: 'a,
    {
        self
    }
}

trait Sub: Super {}

fn upcast(obj: Arc<dyn Sub>) -> Arc<dyn Super> {
    obj.as_dyn_super()
}
```

This compiles and works. And whoever implements the `Super` trait does not need to do anything, if the type is sized.

The downside is that `AsDynSuper` is not automatically implemented for DSTs that implement `Super`. If you want to implement `Super` for a DST, then you need to implement `AsDynSuper` and `panic!` in the implementation of `as_dyn_super`, since you cannot create a trait object. This is inconvenient, but not an issue for many use cases.

And because I am such a huge fan of macros, I created the [`as-dyn-trait`](https://crates.io/crates/as-dyn-trait) crate that solves this problem automatically for your traits:

```rust
#[as_dyn_trait]
trait Super {}

trait Sub: Super {}

fn upcast(obj: Arc<dyn Sub>) -> Arc<dyn Super> {
    obj.as_dyn_super()
}
```

## Closing

I understand now why upcasting a trait object in Rust is problematic, and I found a workaround for my use case. On the way, I also explored traits, generics and DSTs in more detail.

If you think any part of this article is confusing, misleading or even incorrect, please [file an issue](https://github.com/brain0/articles/issues) or [open a pull request](https://github.com/brain0/articles/pulls) on GitHub.

Thanks for reading!
