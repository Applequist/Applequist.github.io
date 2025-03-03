---
title: 'Exploring Rust - Traits (Part 1)'
description: 'A serie about the rust programming language.'
pubDate: 'Feb 28 2025'
---

What are Rust traits and what are they used for ?

The short answer is traits are used for 3 things:
* to constrain type parameters in generic programming
* to be able to treat different types the same using a common *interface*.
* to do type metaprogramming

In this post we are going to explore the first use case.

## The simplest generic function you can possibly write

To understand what we mean by "to constrain type parameters", think about
the simplest generic function that takes a generic parameter.

Let's call it `simple`:

```rust, ignore
fn simple<T>(t: T) -> ?? {
  ???
}
```

The function's return type could be anything as far as we know. But
let's assume that the function we are interested in returns something
useful, eg not a constant.

But how could you write the function body? You don't know anything
about the argument `t` and the function body **must** work for any
possible `T` we throw at it!

The truth is that the only thing you can do with something you know nothing
about is just to return it and so, the only such function we could write is:

```rust
fn id<T>(t: T) -> T {
    t
}
```

That's it.

To do something more useful, we need to be able to tell the compiler what a
*valid* type argument for our generic function is.
And by valid, I mean, if we intend to call a method with the signature `foo(&self) -> Bar`
on `t` in the function body, well, `T` better have such a method.

Traits are a way to tell the compiler about such requirements on type parameters.

Note: As we will see it is not enough for a type to have all the methods with the same
signatures as those listed in a trait. The type has to **implement** the trait.
There is no duck-typing in rust.

## Traits 101

Here what [__The Rust Reference__](https://doc.rust-lang.org/stable/reference/items/traits.html)
says about traits:

> A trait describes an abstract interface that types can implement.
> This interface consists of *associated items*, which come in three varieties:
> * functions
> * types
> * constants

Let's say we want to write a generic function what will log the unique id of its
argument.

Using a trait `HasIntId` we can express the requirement for a method that returns a unique
integer id and using a *trait bound* `T: HasIntId` we can constrain the generic type parameter
of our function to have an implementation of the `id` method:

```rust
pub trait HasIntId {
  fn id(&self) -> u64;
}

pub fn log_id<T: HasIntId>(t: T) {
    println!("{}", t.id());
}
```

Let see which types would work with `log_id`:

```rust, ignore
struct Foo {
  id: u64,
}

impl Foo {
  fn id(&self) -> u64 {
    self.id
  }
}

struct Bar {
  id: u64,
}

impl HasIntId for Bar {
  fn id(&self) -> u64 {
    self.id
  }
}

fn main() {
  let bar = Bar { id: 1 };
  log_id(bar);
  let foo = Foo { id: 1 };
  log_id(foo); // <- compilation error here!
}
```

So although `Foo` has a method `id` whose signature is matching
the `HasIntId::id` method, the compiler complains that trait `HasIntId`
is not implemented by `Foo`.

There is no duck-typing in rust! You need to *declare* that `Foo` implements
`HasIntId::id()`.

Ok. So that's it? Traits are just java interface ?!

Yes. In a way. But using the other types of traits associated items make them much more
powerful.

But first, let's also get generic out of the way: yes, trait can be generic!
The typical example is the `From<T>` trait used for infallible value-to-value conversion:

```rust
trait From<T> {
  fn from(t: T) -> Self;
}
```

`Self` above refers to the type that implements the trait.

So now we can also have types whose ids are not integer:

```rust, ignore
trait HasId<I> {
  fn id(&self) -> I;
}

fn log_id<I, T: HasId<I>>(t: T) {
  println!("{}", t.id()); // <- compilation error here
}
```

Ooops... We got a problem! But the compiler has our back and tell us
what the probem is.

We fail to express the full requirement on `T` from our method `log_id`!
Indeed our `log_id` method will print the id of type `I`, which then
must implement the `Display` trait.

We can fix this as follow:

```rust
use std::fmt::Display;

trait HasId<I> {
  fn id(&self) -> I;
}

fn log_id<I: Display, T: HasId<I>>(t: T) {
  println!("{}", t.id());
}
```

But there are some more fundamental problems here as demonstrated in the example below:
* a type could implement `HasId<T>` mutliple time with different `T` which doesn't make
  sense. A type should only have one id whose type is then unique.
* the call site could force a type of id it normally has no saying in.
  The type of id is solely the responsability
  of the type implementing the trait.


```rust
use std::fmt::Display;

trait HasId<I> {
  fn id(&self) -> I;
}

struct Foo;

impl HasId<char> for Foo {
  fn id(&self) -> char {
    'a'
  }
}

impl HasId<u64> for Foo {
  fn id(&self) -> u64 {
    1
  }
}
```

The way to fix this is to use a trait **associated type** instead of a generic type parameter:
An associated type is a type alias associated to a type the same way as other **associated items**
(methods, consts) and it is defined by the type implementing the trait.
An associated type can also have a trait bound to further constrain how it can be defined.

The example below shows that it fixes all our problems:
* the trait can only be implemented once
* the type of id is determined only by the implementing type.
* the type of id **must** implement `Display`


```rust, ignore
use std::fmt::Display;

trait HasId {
  type Id : Display;

  fn id(&self) -> Self::Id;
}

fn log_id<T: HasId>(t: T) {
  println!("{}", t.id());
}

struct Foo {
  id: u64,
}

impl HasId for Foo {
  type Id = u64;

  fn id(&self) -> Self::Id {
    self.id
  }
}

struct FooId(u64);

// This conflict with the previous implementation 
// But if it were the only implementation, problem would be below
impl HasId for Foo {
  type Id = FooId; // <- problem here: FooId does not implement Display

  fn id(&self) -> FooId {
    FooId(self.id)
  }
}
```

As a last example, we can also constrain our `log_id` function to only work with
types whose id are `&str`.

Although the associated type is defined by the type implementing the trait, the
user of the trait can still decide which trait implementation he is willing/able
to deal with:

```rust
use std::fmt::Display;

trait HasId {
  type Id : Display;

  fn id(&self) -> Self::Id;
}

fn log_id<'a, T>(t: T) where T: HasId<Id = &'a str> {
  println!("{}", t.id());
}

struct Foo<'a> {
  id: &'a str
}

impl<'a> HasId for Foo<'a> {
  type Id = &'a str;

  fn id(&self) -> Self::Id {
    self.id
  }
}

struct Bar {
  id: u64,
}

impl HasId for Bar {
  type Id = u64;

  fn id(&self) -> Self::Id {
    self.id
  }
}

fn main() {
  let foo = Foo { id: "foo" };
  log_id(foo);

  let bar = Bar { id: 1 };
  log_id(bar); // (1)
}
```

In (1) the compiler complains: 

> type mismatch resolving `<Bar as HasId>::Id = &str`

## Next level traits: Generic Associated type or GAT

Since version 1.62, associated type can also have type paramater of their own!

Why would you want to do that?

Well, this allows you to express some cool things for your types!

My favourite example is that you can define traits akin to Haskell typeclass.
For example you can define a `Functor` Trait.

For a quick recall, a functor is a type that *contains* values that can be mapped
over. This definition is terrible but it is the best I can come up with. In Haskell
that would be expressed by a typeclass that would look like this:

```haskell, ignore
class Functor f a
  fmap :: (a -> b) -> f a -> f b
```

Note the 2 letters `f a` after the name of the typeclass? You can think of `f` as
a type parameter for the "type that contains..." and `a` as a type parameter for the
type of the contained values. Finally `b` is the type of the contained values in the
functor once it has been mapped over.

The idea of functor is that the structure/container does not change, only the value
inside.

How would you *translate* that in a Rust trait. We could try something like this:

```rust
trait Functor<A> {
  fn fmap<B, M: FnMut(A) -> B>(self, m: M) -> Self;
}

```

Let me explain: `A`, `B` play the same role as `a` and `b` in the Haskell typeclass
and `M` is a type that look like a function from `A` to `B`.
And `Self` kinda plays the same role as `f` in the type typeclass.

Let's try to implement that for, say `Vec<A>`:

```rust, ignore
trait Functor<A> {
  fn fmap<B, M: FnMut(A) -> B>(self, m: M) -> Self;
}

impl<A> Functor<A> for Vec<A> {
  fn fmap<B, M: FnMut(A) -> B>(self, m: M) -> Self {
    self.into_iter().map(m).collect()
  }
}
```

Spoiler alert: this does not compile!

Indeed, the return type, `Self` is `Vec<A>` but `self.into_iter().map(m).collect()` 
will return an `Vec<B>`.

It would be nice if `Self` (or something else) could somehow refer to the higher kinded type `Vec` instead
of the type `Vec<A>`, this way we could write the trait as follow:

```rust, ignore
trait Functor<A> {
  fn fmap<B, M: FnMut(A) -> B>(self, m: M) -> Self<B>;
}
```

to capture the fact that the *container* type `Self` remains the same but that
the type of the *contained* value has changed during `fmap` from `A` to `B`.

Unfortunately we can't...

But... Enter GAT!

So, if we can't have `Self<B>`, we can however introduce a generic associated type
`F<B>` that stands in for it in the example above:

```rust
trait Functor {
  type A;
  type F<B>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B>;
}

impl<T> Functor for Vec<T> {
  type A = T;
  type F<B> = Vec<B>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B> {
    self.into_iter().map(m).collect()
  }
}
```

Tada! The important part here is `type F<B> = Vec<B>;`.
The `fmap` implementation for `Vec<T>` maps `Self ~ Vec<T>` to `Self::F<B> ~ Vec<B>`.
That's what we wanted.

So let's also implement that for `Result<T, E>` and `Option<T>`:

```rust
trait Functor {
  type A;
  type F<B>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B>;
}

impl<T> Functor for Vec<T> {
  type A = T;
  type F<B> = Vec<B>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B> {
    self.into_iter().map(m).collect()
  }
}

impl<T, E> Functor for Result<T, E> {
  type A = T;
  type F<B> = Result<B, E>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B> {
    self.map(m)
  }
}

impl<T> Functor for Option<T> {
  type A = T;
  type F<B> = Option<B>;

  fn fmap<B, M: FnMut(Self::A) -> B>(self, m: M) -> Self::F<B> {
    self.map(m)
  }
}

fn map<F, B, M>(m: M, f: F) -> F::F<B>
where
    F: Functor,
    M: FnMut(F::A) -> B
{
    f.fmap(m)
}

fn double(n: i32) -> i32 { 2 * n }

assert_eq!(vec![1, 2, 3, 4].fmap(double), vec![2, 4, 6, 8]);
assert_eq!(None.fmap(double), None);
assert_eq!(Some(1).fmap(double), Some(2));
assert_eq!(Err("Boom!").fmap(double), Err("Boom!"));
// For this one the compiler need a little hint from our friend the turbo :)
assert_eq!(Ok::<i32, &str>(5).fmap(double), Ok(10));
```

